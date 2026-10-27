# Changelog

## 2026-10-27: Desktop sub-sessions

- Tasks now run as **sub-sessions under the orchestrator**, started with Claude Desktop's `mcp__ccd_session__start_session` (`use_worktree: true`, `context: "fresh"`, `initiation: "user_asked"`).
- A sub-session's final message reaches the orchestrator automatically as a turn-end note, so the contract makes that message the structured report.
- The orchestrator answers sub-sessions with `SendMessage` to their `local_…` id, and watches them with `get_session` and `list_events`.
- Separate `claude --bg` sessions are now only a fallback for terminal CLI sessions, used with the user's approval.

## 2026-10-27: CLI background sessions (superseded)

- Tasks ran as separate `claude --bg -n -w` sessions instead of Agent-tool subagents, and sub-sessions were allowed to start their own subagents.
- A live probe established these facts, still documented under the CLI fallback:
  - `claude --bg` requires a trusted workspace.
  - `command claude` bypasses shell wrappers.
  - A new session can't receive messages for about 15 s.
  - `claude stop` and `claude rm` need the short id.
  - Idle notices wake an idle orchestrator.

## 2026-10-27: Initial version (superseded)

- The orchestrator delegated tasks to Agent-tool subagents with `isolation: "worktree"`.
- The core rules introduced here are still in place:
  - strict TDD with observed RED/GREEN/REFACTOR evidence
  - mandatory regression guards
  - the orchestrator as the single gateway for questions and approvals
  - branch-based verification
  - an integration worktree with a full-suite run
  - approval before merging into the user's branch
- Also added: the `/ocd` command wrapper, MIT license, and three eval prompts.
