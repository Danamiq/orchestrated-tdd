# orchestrated-tdd

A Claude Code skill that turns the main conversation into a development orchestrator. The orchestrator:

- plans a feature request and splits it into task contracts
- starts each task as a *sub-session* under the current session (Claude Desktop `start_session`), in its own git worktree
- lets each sub-session start its own subagents, while requiring strict TDD (observed RED → GREEN → REFACTOR) and regression tests
- relays all sub-session questions and approvals to you, so you only ever talk to the main conversation; you can still open any sub-session in the Desktop app
- merges the task branches in a separate integration worktree, runs the full suite there, and asks before landing anything on your branch

## Install

```bash
git clone https://github.com/Danamiq/orchestrated-tdd ~/.claude/skills/orchestrated-tdd
mkdir -p ~/.claude/commands
cp ~/.claude/skills/orchestrated-tdd/commands/ocd.md ~/.claude/commands/ocd.md   # optional /ocd shortcut
```

Restart Claude Code, or start a new session, to load the skill.

## Use

```
/ocd add CSV export to GET /reports/:id/export and a `reports export <id> --format csv` CLI command
```

`/orchestrated-tdd ...` works too, and Claude may pick the skill on its own for multi-file features you ask to have built test-first.

## Layout

| Path | Purpose |
|---|---|
| `SKILL.md` | Orchestrator workflow, approval boundaries, final report format |
| `references/task-contract.md` | Prompt template given to each sub-session |
| `references/tdd-cycle.md` | TDD phases A–F, regression-guard checklist, stack detection |
| `references/orchestration.md` | Starting sub-sessions, turn-end reports, messaging, worktrees, retries, conflicts, integration, CLI fallback |
| `commands/ocd.md` | `/ocd` slash-command wrapper |
| `evals/` | Test-case prompts for `claude plugin eval`. They need a target repo; they are not run in CI. |

## Requirements and limitations

- Run it from a Claude Code session in the **Claude Desktop app**: sub-sessions use the Desktop session tools (`mcp__ccd_session__start_session`, `get_session`, `list_events`). In a terminal CLI session those tools don't exist. The skill says so and offers a fallback (separate `claude --bg` sessions), which you must approve.
- The target project must be a git repository. Uncommitted changes are invisible to worktrees.
- A permission prompt inside a sub-session can't be relayed; open that sub-session in the Desktop app to answer it.
- Messages between sessions arrive at the receiver's next tool round or turn, not as live interrupts.
- See `references/orchestration.md` → "Known limitations".

## License

MIT
