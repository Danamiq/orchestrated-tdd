# orchestrated-tdd

A Claude Code skill that turns the main conversation into a development orchestrator. The orchestrator:

- plans a feature request and splits it into task contracts
- delegates each task to a subagent working in its own git worktree
- requires every subagent to follow strict TDD (observed RED → GREEN → REFACTOR) and add regression tests
- relays all subagent questions and approvals to you, so you only ever talk to the main conversation
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
| `references/task-contract.md` | Prompt template given to each subagent |
| `references/tdd-cycle.md` | TDD phases A–F, regression-guard checklist, stack detection |
| `references/orchestration.md` | Worktrees, messaging limits, retries, conflicts, integration, user access |
| `commands/ocd.md` | `/ocd` slash-command wrapper |
| `evals/` | Test-case prompts for `claude plugin eval`. They need a target repo; they are not run in CI. |

## Requirements and limitations

- Claude Code with the Agent tool's worktree isolation and `SendMessage`. Written against 2.1.x.
- The target project must be a git repository; uncommitted changes are invisible to worktrees.
- Subagent questions arrive at the subagent's next tool round or when it finishes, not as live interrupts.
- See `references/orchestration.md` → "Known limitations".

## License

MIT
