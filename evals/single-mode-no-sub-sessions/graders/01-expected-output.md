---
type: llm
focus: trace
weight: 1
---

The result matches this description: The session does the fix itself without calling start_session or messaging any sub-session; works in its own worktree/branch rather than the user's branch; writes a test reproducing the apostrophe 500 and shows it failing for the expected reason before the fix; fixes it with a parameterized query or equivalent; keeps the test as a regression guard; flags the input-handling/security tier versus its current model; runs the full suite; reports with the final-report template; and asks before merging.
