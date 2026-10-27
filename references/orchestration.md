# Orchestration mechanics

These notes were checked against Claude Code 2.1.x. If a tool or flag named here is missing in the running session, use the closest one that is available and tell the user what changed. Don't pretend the missing one exists.

## Contents
- Mechanisms
- Messaging: what is and isn't live
- Worktree base
- Stacked tasks
- Concurrency and file ownership
- Retries and failures
- Conflicts
- Integration
- User access to a subsession
- Persistence across sessions
- Known limitations

## Mechanisms

| Need | Mechanism |
|---|---|
| Spawn a subsession | `Agent` tool, `subagent_type: "general-purpose"`, `isolation: "worktree"`, `name: "ocd-<task>"`, prompt = contract. Runs in the background, and you are notified when it finishes. |
| Answer a question or resume a finished/blocked agent | `SendMessage` with `to: "<agent name>"`. A send to a completed agent resumes it from its transcript with full context. |
| Subagent → orchestrator mid-run | The subagent calls `SendMessage` with `to: "main"`. |
| Stop a runaway or conflicting agent | `TaskStop` with the agent's name. |
| See what is alive | `ListAgents` (your subagents and other local sessions). |
| Ask the user | `AskUserQuestion` for discrete options; plain text otherwise. |
| User-attachable session (opt-in) | CLI background session; see "User access to a subsession". |

Subagents don't inherit your conversation. They inherit only the contract and whatever the environment loads for every session, such as global CLAUDE.md.

## Messaging: what is and isn't live

- Messages are queued. They are delivered at the receiver's **next tool round**, not as an interrupt. A subagent's question reaches you while it keeps working, or when it ends its turn.
- A subagent can't sit and wait for a reply. If it has nothing else to do, it ends its turn with a BLOCKED report. Your `SendMessage` reply then resumes it. This return-and-resume loop is the dependable approval path; treat mid-run messages as an optimization.
- Never invent a message, a status, or an approval. If you haven't heard from an agent, its status is unknown. Check `ListAgents` once, or wait for the completion notification. Don't poll in a loop.
- Approval relay: you ask the user, the user answers, and you send that exact decision to the agent. If the user hasn't answered, the agent stays BLOCKED on that point.

## Worktree base

The Agent tool's worktree isolation creates a fresh worktree under `.claude/worktrees/` on a new branch. Depending on the `worktree.baseRef` setting, that branch may start from `origin/<default-branch>` rather than the user's local HEAD. The contract's start-up check corrects this: it resets a still-clean fresh worktree to `BASE_SHA`. Worktrees share one object store, so any local commit is reachable.

**Uncommitted changes in the user's tree are invisible to every worktree.** If the tasks depend on them, stop and ask the user to choose:
1. The user commits them (or allows you to make a WIP commit on their branch), and `BASE_SHA` becomes that commit.
2. Proceed from the last commit, if the tasks don't actually need the uncommitted work.
3. Run without worktrees: sequential tasks in the shared tree with strict file ownership. This is the fallback when worktrees can't work.

Never stash, reset, or commit the user's working tree without that approval.

Fresh worktrees have no installed dependencies or build artifacts. Put the project's install command in the contract (e.g. `npm ci`, `uv sync`), which costs time and disk per worktree. That is one more reason to consolidate small tasks.

## Stacked tasks

When task B depends on task A: wait until A is VERIFIED, then set B's base to A's branch HEAD SHA. B's start-up check resets onto it. When integrating, merge A first, then B. Don't start B early against A's unfinished interface. If you need parallelism, agree on the interface in both contracts and give the shared interface file to exactly one owner.

## Concurrency and file ownership

- Each file or interface has exactly one owning task at a time. Worktrees prevent clobbered files at edit time but not merge conflicts later, so overlapping ownership still means sequential work or consolidating the tasks into one subsession.
- Shared hot spots (route tables, DI registries, barrel/index files, lockfiles, migrations, generated code): give each to one task. Alternatively, keep them for yourself and add those lines during integration.
- A subagent that discovers it needs a file it doesn't own reports BLOCKED. You re-assign ownership, sequence the work, or consolidate.

## Retries and failures

Before retrying, look at what already exists:
1. `git log --oneline BASE_SHA..<task-branch>`, plus `git -C <worktree> status` if the worktree still exists (uncommitted work lives only there). Keep good partial work.
2. Prefer resuming the same agent (`SendMessage` to its name) with the specific failure. It keeps its context and its worktree.
3. If the agent is wedged, spawn a new one with the **original contract** plus "continue from branch X at SHA Y; already done: ...; remaining: ...". Its start-up check resets to that SHA instead of `BASE_SHA`.
4. Doing it yourself is fine for small remainders. Do it in that worktree, not in the user's tree.
5. Keep each task's test evidence in the ledger across retries.
6. Two failed attempts on the same point → change approach or ask the user. Don't loop.

## Conflicts

Two tasks with incompatible changes (merge conflict, divergent interface, duplicated logic):
1. Stop integration. `TaskStop` any agent still editing the affected area.
2. Decide the resolution: decide it yourself if it is mechanical; ask the user if it changes behavior or scope; otherwise give it to one owner with a corrected contract.
3. Never resolve by silently dropping one side's changes.
4. After resolving, re-run both tasks' tests and the full suite in the integration worktree.

## Integration

Run only when every task is VERIFIED and no agent is running (`ListAgents`).

```bash
# from the repo root; FEATURE is a short slug
git worktree add -b ocd/FEATURE .claude/worktrees/ocd-integrate-FEATURE BASE_SHA
cd .claude/worktrees/ocd-integrate-FEATURE
git merge --no-ff <task-branch-1>      # dependency order
git merge --no-ff <task-branch-2>
# install deps if needed, then the FULL verification set: all tests, lint, typecheck, build
```

- If `.claude/worktrees/` shows up as untracked in the user's `git status`, never commit it. Mention it to the user; they may want it in `.gitignore`.
- Landing needs user approval. On approval, from the user's checkout: if their branch is still at `BASE_SHA`, run `git merge --ff-only ocd/FEATURE`; otherwise run `git merge ocd/FEATURE`. If they have uncommitted changes that overlap the merged files, ask first.
- Cleanup also needs approval: `git worktree remove <path>` and `git branch -d <branch>`. Use `-d`, not `-D`; an unmerged branch refusing deletion is a signal, not an obstacle.

## User access to a subsession

Offer whichever fits the request:

1. **Relay (default).** The user tells you what to say and you `SendMessage` it to the agent. The agent keeps its full context.
2. **Work in the task's worktree directly.** Make sure the agent is idle or stopped (`TaskStop`), then give the user the exact command: `cd <worktree-path>` and `claude`. That starts a new session in the same files and branch. It does **not** carry the subagent's conversation. When the user returns, re-read the worktree (`git log`, `git status`, run the tests) before continuing, and fold their changes into the ledger.
3. **Attachable background sessions (opt-in, when the user wants to drop into live tasks).** Launch the task as a CLI background session instead of an Agent subagent, for example `claude --bg -n ocd-TASK -w ocd-TASK "<contract text>"`. Check `claude --help` first: confirm the flags exist and see how the prompt is passed. The user attaches with `claude attach ocd-TASK` (← returns to agent view; the session keeps running). Track it with `claude agents --json` and `claude logs ocd-TASK`. Message it via `ListAgents` + `SendMessage`, and use `notify_when_idle: true` instead of polling. Caveats: the harness doesn't notify you when a background session finishes, so you depend on idle notices. If its permission mode differs from yours, it may hold your messages for the user's approval or drop them without telling you. The orchestrator stays responsible for verification and integration either way.

## Persistence across sessions

Named agents can be resumed within this orchestrator session. If this session ends, the worktrees and branches survive (`git worktree list`, `git branch --list "ocd/*"` plus the task branch names in the ledger), but the agent conversations may not be reachable. For long runs, write the ledger (task, branch, worktree, status, evidence) to the session scratchpad or the memory directory at each milestone. A new session can then rebuild state from git plus the ledger.

## Known limitations

- No true interrupt: questions arrive at tool-round or turn boundaries.
- Worktrees isolate edits, not semantics. Two correct branches can still break together, which is why integration verification is mandatory.
- The base of a worktree depends on settings; the start-up check guards it.
- Only committed work reaches worktrees.
- Concurrency multiplies disk, dependency installs, and token cost; cap parallel agents.
- Automatic triggering depends on the skill description. `/ocd` or `/orchestrated-tdd` always invokes it explicitly.
