---
type: llm
focus: trace
weight: 1
---

The result matches this description: Orchestrator detects test/lint commands, plans two tasks (HTTP endpoint, CLI) with disjoint file ownership and a shared serializer owned by one task, launches two claude --bg subsessions (-w worktrees, same permission mode, contract plus TDD cycle inlined) and subscribes to their idle notices, verifies RED/GREEN evidence per task, merges into an integration worktree, runs the full suite there, and asks before merging into the user's branch.
