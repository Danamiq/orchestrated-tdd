---
description: Orchestrated TDD fix - in the original /ocd chat, iterate on the feature by reusing its existing sub-sessions (no new child sessions)
argument-hint: what to fix
---

Invoke the `orchestrated-tdd` skill with the Skill tool and follow its "Fix mode (`/ocd-fix`)" section for this fix request. This only works in the orchestrator chat that ran `/ocd` for the feature; if this chat started no `ocd:` sub-sessions, stop and tell me to run it there. Reuse the sub-sessions that built the feature; do not start new child sessions unless I explicitly approve it.

$ARGUMENTS
