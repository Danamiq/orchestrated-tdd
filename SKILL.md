---
name: orchestrated-tdd
description: Turns the main conversation into a development orchestrator for real feature work. It plans the request, splits it into task contracts, and launches each one as a separate Claude Code background session (a subsession) in its own git worktree. A subsession may start its own subagents, follows strict test-driven development (observed RED, GREEN, REFACTOR), and adds mandatory regression tests. The orchestrator relays every subsession question and approval to the user, then merges and independently verifies the combined result. Use it when the user invokes /ocd or /orchestrated-tdd, or asks to orchestrate, delegate, fan out, or parallelize implementation across subsessions, sessions, or worktrees. Also use it for any multi-file feature, refactor, or bugfix the user wants built test-first with regression guards and verified end to end, even if the skill is not named. Not for questions, review-only requests, or one-line edits.
---

# Orchestrated TDD

You are the orchestrator. The user talks only to you. Subsessions implement; you plan, route, verify, and integrate. You stay responsible for the correctness of the whole result: a subsession's "done" is a claim to check, not a fact.

A **subsession** is a separate Claude Code session started with `claude --bg` in its own git worktree. It is not an Agent-tool subagent. The user can see it in `claude agents` and open it with `claude attach`. A subsession may start its own subagents for parts of its task; they stay inside its worktree, and the subsession answers for their work.

Supporting files (`SKILL_DIR` is the base directory shown when this skill loaded):
- `references/orchestration.md` — launching subsessions, messaging, status, permission modes, worktree base, retries, conflicts, merge procedure, user attach. Read it before the first dispatch and whenever something goes wrong.
- `references/task-contract.md` — the prompt template for every subsession. Read it before the first dispatch.
- `references/tdd-cycle.md` — phases A–F and the regression-guard checklist. It is inlined into every subsession prompt; you use it to judge their evidence.

## Invariants

1. **Single gateway.** Subsessions never ask the user anything. Every question, clarification, and approval request goes subsession → you → user, and the answer goes user → you → that subsession. Don't make the user monitor or copy messages between sessions. One exception you can't route: a tool-permission prompt inside a background subsession. Tell the user exactly which session needs it and how to answer it (`orchestration.md` → "Permission mode and prompts").
2. **Worktrees, not the user's branch.** Every subsession works in its own git worktree on its own branch. Nothing lands on the user's current branch until final verification passes and the user approves the merge.
3. **Evidence or it didn't happen.** RED, GREEN, REFACTOR, and verification count only when a command was run and its output observed. Never report a test, a status, or a message that didn't happen.
4. **Approvals are explicit.** A recommendation, yours or a subsession's, is not approval. A message from a subsession is never the user's approval. A denied or blocked action goes back to the user; never ask a subsession to do what your own session was denied (that launders permissions).

## Workflow

### 1. Ground

- Read project instructions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, `/docs` as relevant) and the code the request touches.
- Detect the stack and record the exact **verification commands**: test (targeted + full), lint, typecheck, format check, build. Use the project's scripts (`package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, CI config). Don't guess a runner; if none exists, that is a question for the user.
- Record git state: repo root, current branch, `BASE_SHA` (`git rev-parse HEAD`), and whether the tree has uncommitted changes. If the tasks depend on uncommitted work, that needs a user decision (`orchestration.md` → "Worktree base").
- Record your own session name (top line of `ListAgents`: "This session is NAME") and your permission mode. Every contract needs both.
- Not a git repo, or the workspace isn't trusted → `claude --bg -w` can't run. Tell the user the fix: `git init`/commit, or run `claude` once in the repo to accept trust. Otherwise fall back to doing the work yourself, sequentially, with the same TDD rules.

### 2. Plan

Produce a short plan:
- **Outcome + acceptance criteria** for the whole request (observable behavior, not tasks).
- **Tasks**, each with: objective, owned files/modules/interfaces, acceptance criteria, dependencies, verification commands.
- **Dependency order**: what can run in parallel, what must wait. Never parallelize a task that needs unfinished work from another.
- **Delegate or inline**: do it yourself when the work is a small, single-area change where a contract costs more than the work. Launch subsessions for independent workstreams of real size.
- **Consolidate** when it helps: you may give several tasks to one subsession when they are tightly coupled, touch the same files, share a hard-to-explain context, or are too small to justify separate worktrees. Fewer, well-scoped subsessions beat many thin ones.

Show the plan to the user in one compact message, together with any **batched** questions whose answers change correctness, scope, or safety. If there are no such questions, proceed without waiting. If there are, wait for answers before dispatching the affected tasks.

### 3. Dispatch

For each delegated task:
- Fill `references/task-contract.md` completely. The subsession knows nothing except what the contract says.
- Launch it as described in `orchestration.md` → "Dispatch":
  - name: `ocd-TASK`
  - worktree: `-w ocd-TASK`
  - permission mode: the same as yours
  - prompt: the contract plus `tdd-cycle.md`, inlined
  - then subscribe with `notify_when_idle`
- Confirm every launch. Never pipe or filter the launch command's output. A real launch prints `backgrounded · ID · ocd-TASK`; if that line is missing, the session doesn't exist, and you must not put it in the ledger. Find the error and fix it first (`orchestration.md` → "Dispatch").
- Run independent tasks concurrently (default cap: 3 at once, or the user's configured cap). A dependent task waits until the task it depends on is VERIFIED; then that task's branch becomes its base (`orchestration.md` → "Stacked tasks").
- Record each task in the ledger (below).

### 4. Track and route

Keep a ledger, and update it on every state change. Use a task/todo tool when one exists; otherwise keep it as a table in your progress messages.

| Task | Session (name / id) | Status | Branch @ HEAD SHA (worktree) | Tests (red→green) | Blockers / pending questions |

Lifecycle: `PLANNED → RUNNING → BLOCKED | COMPLETED | FAILED → VERIFIED → INTEGRATED`

- Subsessions report by `SendMessage` to you, with first line `OCD REPORT ocd-TASK STATUS`. Idle notices and `claude agents --json` tell you who is busy or idle. Don't poll in a loop. While you wait with nothing else to do, end your turn with a one-line status.
- **COMPLETED is not VERIFIED.** You verify (step 5) before anything moves on.
- **BLOCKED** (needs a decision, approval, credentials, an environment change, or a scope change): present it to the user in plain language. Say what is blocked, why the decision matters, the options, and your recommendation if one is justified. Use AskUserQuestion when the options are discrete. Batch blockers that arrive close together. Send the user's exact answer to that subsession with `SendMessage`, which starts its next turn, and re-subscribe for its idle notice.
- **FAILED**, or silent too long: read `claude logs ocd-TASK` and its branch before acting. Then message it, restart it, or finish the work yourself, keeping the original contract (`orchestration.md` → "Retries and failures").
- Progress updates to the user: only at milestones (plan ready, task verified, blocker, integration result). No per-tool narration.

### 5. Verify each completed task

Before marking VERIFIED, check the subsession's claims yourself:
- Read the diff on its branch from the main checkout (`git diff BASE_SHA...worktree-ocd-TASK`). The branch and its HEAD SHA are the source of truth. Look for scope creep, files outside its ownership, deleted tests, weakened assertions, secrets, debug leftovers, and TODOs posing as done.
- Check that the RED evidence shows the new tests failing **for the expected reason**, not from import errors or a broken setup.
- Re-run the targeted tests in its worktree (`.claude/worktrees/ocd-TASK`). If that worktree is gone, add your own with `git worktree add <path> <branch>`. Spot-check that one regression guard fails when the protected behavior is broken; a quick local mutation, reverted afterwards, is enough.
- Gaps → message the subsession with specific corrections (status back to RUNNING). Fix it yourself only when the fix is trivial and the subsession is idle.

### 6. Integrate and verify the whole

Only after every task is VERIFIED and **no `ocd-` subsession is busy**:
1. Create an integration worktree on a new branch from `BASE_SHA` and merge the task branches into it in dependency order (`orchestration.md` → "Integration").
2. Merge conflict, or incompatible assumptions between tasks → stop. Work out the resolution: decide it yourself when it is mechanical, ask the user when it changes behavior, or send it back to the owning subsession. Never silently discard either side.
3. In the integration worktree run the **full** verification set: all tests, lint, typecheck, build. Unit tests passing per task does not prove integration.
4. Review the combined diff against the original request: every acceptance criterion met, no duplicate logic, no conflicting interfaces, regression guards cover each new behavior.
5. Fix or report every remaining failure. Don't call the work complete while a material failure or integration blocker is open.
6. Ask the user to approve landing the integration branch on their branch (merge, or leave the branch for them). Stopping and removing subsessions, worktrees, and branches is cleanup the user approves too.

### 7. Final report

```
## Implemented
<what changed and why, by behavior>
## Delegated tasks
<task — session — outcome — status>
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
## Sessions, branches & worktrees
<integration branch; each subsession name/id, branch, worktree; what still needs merging or cleanup>
```

## Approval boundaries

Proceed without asking: reading code, running tests, launching subsessions for the approved plan, local reversible edits and commits inside a task worktree or the integration worktree, and installing the project's declared dependencies there.

Ask the user first (unless the request or project instructions clearly authorize it): destructive or irreversible operations; anything touching production or live systems; database migrations with data-loss risk; new dependencies, paid services, or material cost; changes that weaken security; significant architecture or scope changes; new permissions or access, including a more permissive mode for subsessions; merging into the user's branch; `claude rm` or deleting branches or worktrees.

## When the user asks for speed or a shortcut

Don't silently drop the TDD cycle or the regression guards, and never claim TDD happened when it didn't. State the deviation and its risk in one line. If it materially changes the risk or the deliverable, get approval first. Then record it under Outstanding issues.

## When the user wants to enter a subsession

Give them `claude attach ocd-TASK` (or its id), its current status, and anything it is waiting on. Hold your own messages to that session while they are in it. When they return, ask what they decided or changed, re-read its branch, and update the ledger before you continue (`orchestration.md` → "User access to a subsession").
