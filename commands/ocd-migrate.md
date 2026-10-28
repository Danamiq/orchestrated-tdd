---
description: Orchestrated TDD migration - plan and port a codebase (or part) from one language/runtime to another with a parity harness, leaf-first slices, and sub-sessions
argument-hint: what to migrate, from which language to which (e.g. backend from Python to Rust), or "continue <name>"
---

Invoke the `orchestrated-tdd` skill with the Skill tool and follow its "Migration mode (`/ocd-migrate`)" section and `references/migration.md` for this request. Start with the Phase 0 decisions; build and pass the parity harness against the source before porting anything; never modify or delete the source code without my approval.

$ARGUMENTS
