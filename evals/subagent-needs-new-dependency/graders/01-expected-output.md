---
type: llm
focus: trace
weight: 1
---

The result matches this description: When a subagent reports it needs a new dependency (e.g. express-rate-limit), the orchestrator pauses that task, presents the choice to the user with options and a recommendation, and only resumes the subagent via SendMessage after the user answers; it never installs the dependency on the subagent's say-so.
