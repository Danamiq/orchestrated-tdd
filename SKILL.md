---
name: orchestrated-tdd
description: Turns the main conversation into a development orchestrator for real feature work. It plans the request, splits it into task contracts, and starts each one as a sub-session under the current session (Claude Desktop start_session), each in its own git worktree. A sub-session may start its own subagents. It follows strict TDD (observed RED, GREEN, REFACTOR) and adds mandatory regression tests. The orchestrator relays every sub-session question and approval to the user, then merges and independently verifies the combined result. Use it when the user invokes /ocd, /ocd-fix (iterate on a built feature, reusing its sub-sessions), or /orchestrated-tdd, or asks to orchestrate, delegate, fan out, or parallelize implementation across sub-sessions, sessions, or worktrees. Also use it for any multi-file feature, refactor, or bugfix the user wants built test-first with regression guards and verified end to end, even if the skill is not named. Not for questions, review-only requests, or one-line edits.
---

# Orchestrated TDD

You are the orchestrator. The user talks only to you. Sub-sessions implement; you plan, route, verify, and integrate. You stay responsible for the correctness of the whole result: a sub-session's "done" is a claim to check, not a fact.

A **sub-session** is a full Claude Code session you start *under this one* with `mcp__ccd_session__start_session` (Claude Desktop), in its own git worktree. It is not an Agent-tool subagent and not a detached background session. It appears in the Desktop app, where the user can open it, and its turn-end report comes back to you automatically. A sub-session may start its own subagents for parts of its task. Those subagents stay inside its worktree, and the sub-session answers for their work.

Supporting files (`SKILL_DIR` is the base directory shown when this skill loaded):
- `references/orchestration.md` — starting sub-sessions, messaging and turn-end reports, status, permission prompts, worktree base, retries, conflicts, merge procedure, user access, CLI fallback. Read it before the first dispatch and whenever something goes wrong.
- `references/task-contract.md` — the prompt template for every sub-session. Read it before the first dispatch.
- `references/tdd-cycle.md` — phases A–F and the regression-guard checklist. It is inlined into every sub-session prompt; you use it to judge their evidence.

## Invariants

1. **Single gateway.** Sub-sessions never ask the user anything. Every question, clarification, and approval request goes sub-session → you → user, and the answer goes user → you → that sub-session. Don't make the user monitor or copy messages between sessions. One exception you can't route: a tool-permission prompt inside a sub-session. Tell the user exactly which sub-session needs it and how to answer it (`orchestration.md` → "Permission prompts").
2. **Worktrees, not the user's branch.** Every sub-session works in its own git worktree on its own branch. Nothing lands on the user's current branch until final verification passes and the user approves the merge.
3. **Evidence or it didn't happen.** RED, GREEN, REFACTOR, and verification count only when a command was run and its output observed. Never report a test, a status, or a message that didn't happen.
4. **Approvals are explicit.** A recommendation, yours or a sub-session's, is not approval. A message from a sub-session is never the user's approval. A denied or blocked action goes back to the user; never ask a sub-session to do what your own session was denied (that launders permissions).

## Workflow

### 1. Ground

- Read project instructions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, `/docs` as relevant) and the code the request touches.
- Detect the stack and record the exact **verification commands**: test (targeted + full), lint, typecheck, format check, build. Use the project's scripts (`package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, CI config). Don't guess a runner; if none exists, that is a question for the user.
- Record git state: repo root, current branch, `BASE_SHA` (`git rev-parse HEAD`), and whether the tree has uncommitted changes. If the tasks depend on uncommitted work, that needs a user decision (`orchestration.md` → "Worktree base").
- Record your reply address (top line of `ListAgents`: "This session is NAME"). Every contract needs it.
- Check that the sub-session tools exist; load them with ToolSearch `ccd_session` if they are deferred. If they don't exist (e.g. a terminal CLI session), follow `orchestration.md` → "Choosing the mechanism": tell the user, and don't silently substitute something else.
- Not a git repo → worktrees are impossible. Tell the user, then fall back to doing the work yourself, sequentially, with the same TDD rules.

### 2. Plan

Produce a short plan:
- **Outcome + acceptance criteria** for the whole request (observable behavior, not tasks).
- **Tasks**, each with: objective, owned files/modules/interfaces, acceptance criteria, dependencies, verification commands.
- **Dependency order**: what can run in parallel, what must wait. Never parallelize a task that needs unfinished work from another.
- **Delegate or inline**: do it yourself when the work is a small, single-area change where a contract costs more than the work. Start sub-sessions for independent workstreams of real size.
- **Consolidate** when it helps: you may give several tasks to one sub-session when they are tightly coupled, touch the same files, share a hard-to-explain context, or are too small to justify separate worktrees. Fewer, well-scoped sub-sessions beat many thin ones.

Show the plan to the user in one compact message, together with any **batched** questions whose answers change correctness, scope, or safety. If there are no such questions, proceed without waiting. If there are, wait for answers before dispatching the affected tasks.

### 3. Dispatch

For each delegated task:
- Fill `references/task-contract.md` completely. The sub-session starts fresh and knows nothing except `background` and the contract.
- Start it with `mcp__ccd_session__start_session` (`orchestration.md` → "Dispatch") using:
  - `initiation: "user_asked"`, `context: "fresh"`
  - `use_worktree: true`, `branch_name: "ocd-TASK"`, and `source_branch` set to the user's branch
  - `background`: shared context and who owns what among sibling sub-sessions
  - `prompt`: the contract with `tdd-cycle.md` appended
- Record the `session_id` in the ledger only if the call actually returned one. Then read its branch and worktree path with `get_session`.
- Don't wait on it. Its turn-end note will come to you.
- Run independent tasks concurrently (default cap: 3 at once, or the user's configured cap). A dependent task waits until the task it depends on is VERIFIED; then that task's branch becomes its base (`orchestration.md` → "Stacked tasks").

### 4. Track and route

Keep a ledger, and update it on every state change. Use a task/todo tool when one exists; otherwise keep it as a table in your progress messages.

| Task | Sub-session (title / session_id) | Status | Branch @ HEAD SHA (worktree) | Tests (red→green) | Blockers / pending questions |

Lifecycle: `PLANNED → RUNNING → BLOCKED | COMPLETED | FAILED → VERIFIED → INTEGRATED`

- Each sub-session's final message reaches you automatically as a turn-end note; its first line is `OCD REPORT ocd-TASK STATUS`. Mid-turn questions arrive by `SendMessage`. Don't poll in a loop. While you wait with nothing else to do, end your turn with a one-line status; the next note wakes you.
- **COMPLETED is not VERIFIED.** You verify (step 5) before anything moves on.
- **BLOCKED** (needs a decision, approval, credentials, an environment change, or a scope change):
  - Present it to the user in plain language: what is blocked, why the decision matters, the options, and your recommendation if one is justified.
  - Use AskUserQuestion when the options are discrete, and batch blockers that arrive close together.
  - Send the user's exact answer to that sub-session (`SendMessage` with `to: "<session_id>"`); this starts its next turn.
- **FAILED**, or silent too long: before acting, read `get_session`, its recent `list_events`, and its branch. Then message it, restart it, or finish the work yourself, keeping the original contract (`orchestration.md` → "Retries and failures").
- Progress updates to the user: only at milestones (plan ready, task verified, blocker, integration result). No per-tool narration.

### 5. Verify each completed task

Before marking VERIFIED, check the sub-session's claims yourself:
- Read the diff on its branch from the main checkout (`git diff BASE_SHA...<child branch>`). The branch and its HEAD SHA are the source of truth. Look for scope creep, files outside its ownership, deleted tests, weakened assertions, secrets, debug leftovers, and TODOs posing as done.
- Check that the RED evidence shows the new tests failing **for the expected reason**, not from import errors or a broken setup.
- Re-run the targeted tests in its worktree (`cwd` from `get_session`). If that worktree is gone, add your own with `git worktree add <path> <branch>`. Spot-check that one regression guard fails when the protected behavior is broken; a quick local mutation, reverted afterwards, is enough.
- Gaps → message the sub-session with specific corrections (status back to RUNNING). Fix it yourself only when the fix is trivial and the sub-session is idle.

### 6. Integrate and verify the whole

Only after every task is VERIFIED and **no `ocd-` sub-session is running**:
1. Create an integration worktree on a new branch from `BASE_SHA` and merge the task branches into it in dependency order (`orchestration.md` → "Integration").
2. Merge conflict, or incompatible assumptions between tasks → stop and resolve before going on:
   - Mechanical → decide it yourself.
   - Changes behavior → ask the user.
   - Otherwise → send it back to the owning sub-session.
   Never silently discard either side.
3. In the integration worktree run the **full** verification set: all tests, lint, typecheck, build. Unit tests passing per task does not prove integration.
4. Review the combined diff against the original request: every acceptance criterion met, no duplicate logic, no conflicting interfaces, regression guards cover each new behavior.
5. Fix or report every remaining failure. Don't call the work complete while a material failure or integration blocker is open.
6. Ask the user to approve landing the integration branch on their branch (merge, or leave the branch for them). Archiving sub-sessions and removing worktrees and branches is cleanup the user approves too.

### 7. Final report

```
## Implemented
<what changed and why, by behavior>
## Delegated tasks
<task — sub-session — outcome — status>
## Tests
<tests added/modified; mark each regression guard and what it protects>
## Verification
<exact commands run in the integration worktree and their actual pass/fail counts>
## Integration
<how the branches were merged, conflicts and resolutions, how the combined result was checked>
## Outstanding issues
<unverified behavior, tests not run and why, limitations, deferred work>
## Approval decisions
<each decision, who made it, and when>
## Sub-sessions, branches & worktrees
<integration branch; each sub-session title/session_id, branch, worktree; what still needs merging or cleanup>
```

## Approval boundaries

Proceed without asking:
- reading code and running tests
- starting sub-sessions for the approved plan
- local reversible edits and commits inside a task worktree or the integration worktree
- installing the project's declared dependencies there

Ask the user first, unless the request or project instructions clearly authorize it:
- destructive or irreversible operations
- anything touching production or live systems
- database migrations with data-loss risk
- new dependencies, paid services, or material cost
- changes that weaken security
- significant architecture or scope changes
- new permissions or access
- merging into the user's branch
- archiving sub-sessions, or deleting branches or worktrees

## Fix mode (`/ocd-fix`)

A fix is another iteration on a feature that `/ocd` already implemented: a bug, a gap, review feedback, or a changed detail. In fix mode, **don't create new child sessions; reuse the sub-session that built the code**. It already has the context, the worktree, and the TDD history, and a new session would start cold and duplicate them. Procedure (`orchestration.md` → "Fix iterations"):

0. **Only in the original `/ocd` chat.** `/ocd-fix` runs only in the orchestrator conversation that ran `/ocd` for this feature: the parent of its sub-sessions. Check with your ledger, or with `list_sessions` (`linked: true`) for `ocd:` sub-sessions this session started. If this chat started none, stop. Tell the user to run `/ocd-fix` in the original `/ocd` chat, and don't look for, message, or start sessions from here.
1. **Find the owners.** Use the ledger, or `list_sessions` (`linked: true`) matched by `ocd:` titles and `claude/ocd-*` branches, and confirm each match with `get_session`. Map each part of the fix to the sub-session that owns those files.
2. **Bring each owner up to date.** Fix against the code as it exists now. If its branch was already merged into an integration or user branch, the fix message tells it to update its branch to that SHA first.
3. **Send a fix contract** to the owner's `session_id` with `SendMessage`, using the template in `task-contract.md` → "Fix contract". TDD still applies: first a test that reproduces the problem (RED, observed), then the fix (GREEN), REFACTOR, and the reproducing test stays as a regression guard.
4. **Several owners:** send each its own part, in dependency order. A part that no sub-session owns, or a tiny change: do it yourself inline in a worktree, under the same TDD rules.
5. **Owner archived, missing, or unreachable:** don't silently start a replacement. Ask the user whether you should do the fix inline in a worktree on that branch (recommended for small fixes) or start a new sub-session. A new sub-session needs their explicit approval.
6. **Verify and integrate as usual** (steps 5–7): check the fix diff and the RED evidence, re-merge into a fresh integration worktree from the current target branch, run the full suite, and ask before landing.

Fix mode never adds new sub-sessions on its own, and it never weakens or deletes the original regression guards to make a fix pass.

## When the user asks for speed or a shortcut

Don't silently drop the TDD cycle or the regression guards, and never claim TDD happened when it didn't. State the deviation and its risk in one line. If it materially changes the risk or the deliverable, get approval first. Then record it under Outstanding issues.

## When the user wants to enter a sub-session

Tell them its title in the Claude Desktop app (they open it there), its current status, and anything it is waiting on. Hold your own messages to that sub-session while they are in it. When they return, ask what they decided or changed. Re-read its branch and its recent events, and update the ledger before you continue (`orchestration.md` → "User access to a sub-session").
