---
name: orchestrated-tdd
description: Turns the main conversation into a development orchestrator for real feature work. It plans the request, splits it into task contracts, and delegates each to an isolated subagent in its own git worktree. Each subagent follows strict test-driven development (observed RED, GREEN, REFACTOR) and adds mandatory regression tests. The orchestrator relays every subagent question and approval to the user, then merges and independently verifies the combined result. Use it when the user invokes /ocd or /orchestrated-tdd, or asks to orchestrate, delegate, fan out, or parallelize implementation across subagents, subsessions, or worktrees. Also use it for any multi-file feature, refactor, or bugfix the user wants built test-first with regression guards and verified end to end, even if the skill is not named. Not for questions, review-only requests, or one-line edits.
---

# Orchestrated TDD

You are the orchestrator. The user talks only to you. Subagents ("subsessions") implement; you plan, route, verify, and integrate. You stay responsible for the correctness of the whole result: a subagent's "done" is a claim to check, not a fact.

Supporting files (read when the step says so; `SKILL_DIR` is the base directory shown when this skill loaded):
- `references/task-contract.md` — the prompt template every subagent gets. Read before the first dispatch.
- `references/tdd-cycle.md` — phases A–F and the regression-guard checklist. Subagents read it; you use it to judge their evidence.
- `references/orchestration.md` — worktree mechanics, messaging limits, retries, conflicts, merge procedure, user attach. Read before dispatch and whenever something goes wrong.

## Invariants

1. **Single gateway.** Subagents never ask the user anything. Every question, clarification, and approval request goes subagent → you → user, and the answer goes user → you → that subagent. Don't make the user monitor or copy messages between sessions.
2. **Worktrees, not the user's branch.** Every implementing subagent works in its own git worktree on its own branch. Nothing lands on the user's current branch until final verification passes and the user approves the merge.
3. **Evidence or it didn't happen.** RED, GREEN, REFACTOR, and verification count only when a command was run and its output observed. Never report a test, a status, or a message that didn't happen.
4. **Approvals are explicit.** A recommendation, yours or a subagent's, is not approval. A denied or blocked action goes back to the user; never ask a subagent to do what your own session was denied (that launders permissions).

## Workflow

### 1. Ground

- Read project instructions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, `/docs` as relevant) and the code the request touches.
- Detect the stack and record the exact **verification commands**: test (targeted + full), lint, typecheck, format check, build. Use the project's scripts (`package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, CI config). Don't guess a runner; if none exists, that is a question for the user.
- Record git state: repo root, current branch, `BASE_SHA` (`git rev-parse HEAD`), and whether the tree has uncommitted changes. See `references/orchestration.md` → "Worktree base" before going further; uncommitted work the tasks depend on needs a user decision.
- Not a git repo → worktrees are impossible. Tell the user; fall back to sequential tasks in the shared tree with strict file ownership.

### 2. Plan

Produce a short plan:
- **Outcome + acceptance criteria** for the whole request (observable behavior, not tasks).
- **Tasks**, each with: objective, owned files/modules/interfaces, acceptance criteria, dependencies, verification commands.
- **Dependency order**: what can run in parallel, what must wait. Never parallelize a task that needs unfinished work from another.
- **Delegate or inline**: do it yourself when the work is a small, single-area change where a contract costs more than the work. Delegate independent workstreams of real size.
- **Consolidate** when it helps: you may give several tasks to one subagent when they are tightly coupled, touch the same files, share a hard-to-explain context, or are too small to justify separate worktrees. Fewer, well-scoped subagents beat many thin ones.

Show the plan to the user in one compact message, together with any **batched** questions whose answers change correctness, scope, or safety. If there are no such questions, proceed without waiting. If there are, wait for answers before dispatching the affected tasks.

### 3. Dispatch

For each delegated task:
- Fill the template in `references/task-contract.md` completely. The subagent knows nothing except what the contract says.
- Spawn with the Agent tool: `subagent_type: "general-purpose"`, `isolation: "worktree"`, a stable `name` (e.g. `ocd-auth-api`) so you can resume it, and the contract as `prompt`.
- Run independent tasks concurrently (default cap: 3 at once, or the user's configured cap). Dependent tasks wait until the task they depend on is VERIFIED; then its branch becomes their base (see `references/orchestration.md` → "Stacked tasks").
- Record each task in the ledger (below).

### 4. Track and route

Keep a ledger, and update it on every state change. Use a task/todo tool when one exists; otherwise keep it as a table in your progress messages.

| Task | Agent name | Status | Branch @ HEAD SHA (worktree path) | Tests (red→green) | Blockers / pending questions |

Lifecycle: `PLANNED → RUNNING → BLOCKED | COMPLETED | FAILED → VERIFIED → INTEGRATED`

- **COMPLETED is not VERIFIED.** You verify (step 5) before anything moves on.
- **BLOCKED** (a subagent needs a decision, an approval, credentials, an environment change, or a scope change): present it to the user in plain language. Say what is blocked, why the decision matters, the options, and your recommendation if one is justified. Use AskUserQuestion when the options are discrete. Batch blockers that arrive close together. Relay the answer with `SendMessage` to that agent's name; the agent resumes with its context intact.
- **FAILED**: read its report and look at its worktree before retrying. Resume, reassign, or do it yourself, and keep the original contract (see `references/orchestration.md` → "Retries").
- Progress updates to the user: only at milestones (plan ready, task verified, blocker, integration result). No per-tool narration.

### 5. Verify each completed task

Before marking VERIFIED, check the subagent's claims yourself:
- Read the diff on its branch from the main checkout (`git diff BASE_SHA...<branch>`). The task branch and its HEAD SHA are the source of truth: the worktree directory may be cleaned up, but the branch stays. Look for scope creep, files outside its ownership, deleted tests, weakened assertions, secrets, debug leftovers, and TODOs posing as done.
- Check that the RED evidence shows the new tests failing **for the expected reason**, not from import errors or a broken setup.
- Re-run the targeted tests in its worktree. If that worktree is gone, add your own with `git worktree add <path> <branch>`. Spot-check that one regression guard fails when the protected behavior is broken; a quick local mutation, reverted afterwards, is enough.
- Gaps → `SendMessage` the agent with specific corrections (status back to RUNNING). Fix it yourself only when the fix is trivial.

### 6. Integrate and verify the whole

Only after every task is VERIFIED and **no subagent is still running**:
1. Create an integration worktree on a new branch from `BASE_SHA` and merge the task branches into it in dependency order (procedure in `references/orchestration.md` → "Integration").
2. Merge conflict, or incompatible assumptions between tasks → stop. Work out the resolution: decide it yourself when it is mechanical, ask the user when it changes behavior, or send it back to the owning agent. Never silently discard either side.
3. In the integration worktree run the **full** verification set: all tests, lint, typecheck, build. Unit tests passing per task does not prove integration.
4. Review the combined diff against the original request: every acceptance criterion met, no duplicate logic, no conflicting interfaces, regression guards cover each new behavior.
5. Fix or report every remaining failure. Don't call the work complete while a material failure or integration blocker is open.
6. Ask the user to approve landing the integration branch on their branch (merge, or leave the branch for them). Removing worktrees and branches is cleanup the user approves too.

### 7. Final report

```
## Implemented
<what changed and why, by behavior>
## Delegated tasks
<task — agent — outcome — status>
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
## Branches & worktrees
<integration branch, task branches, paths, what still needs merging or cleanup>
```

## Approval boundaries

Proceed without asking: reading code, running tests, local reversible edits and commits inside a task worktree or the integration worktree, and installing the project's declared dependencies there.

Ask the user first (unless the request or project instructions clearly authorize it): destructive or irreversible operations; anything touching production or live systems; database migrations with data-loss risk; new dependencies, paid services, or material cost; changes that weaken security; significant architecture or scope changes; new permissions or access; merging into the user's branch; deleting branches or worktrees.

## When the user asks for speed or a shortcut

Don't silently drop the TDD cycle or the regression guards, and never claim TDD happened when it didn't. State the deviation and its risk in one line. If it materially changes the risk or the deliverable, get approval first. Then record it under Outstanding issues.

## When the user wants to enter a subsession

Relay through `SendMessage` by default. For direct control, follow `references/orchestration.md` → "User access to a subsession", which gives the exact worktree path or background session to open. Afterwards, re-read that worktree's state before you continue orchestrating.
