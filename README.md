# orchestrated-tdd

A Claude Code skill that turns the main conversation into a development orchestrator. The orchestrator:

- plans a feature request and splits it into task contracts
- launches each task as a separate Claude Code background session (`claude --bg`), a *subsession*, in its own git worktree
- lets each subsession start its own subagents, while requiring strict TDD (observed RED → GREEN → REFACTOR) and regression tests
- relays all subsession questions and approvals to you, so you only ever talk to the main conversation; you can still drop into any subsession with `claude attach ocd-<task>`
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
| `references/task-contract.md` | Prompt template given to each subsession |
| `references/tdd-cycle.md` | TDD phases A–F, regression-guard checklist, stack detection |
| `references/orchestration.md` | Launching subsessions, messaging, permission modes, worktrees, retries, conflicts, integration, attach |
| `commands/ocd.md` | `/ocd` slash-command wrapper |
| `evals/` | Test-case prompts for `claude plugin eval`. They need a target repo; they are not run in CI. |

## Requirements and limitations

- Claude Code with background sessions (`claude --bg`, `claude agents`, `claude attach`) and cross-session `SendMessage`. Tested on 2.1.311.
- The target project must be a git repository and a trusted workspace (run `claude` in it once). Uncommitted changes are invisible to worktrees.
- Run the orchestrator in `auto` or `acceptEdits` mode: subsessions inherit the same mode, and a permission prompt in a background session waits until you `claude attach` to answer it.
- Messages between sessions arrive at the receiver's next tool round or turn, not as live interrupts.
- See `references/orchestration.md` → "Known limitations".

## License

MIT
