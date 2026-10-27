---
tags: [skill-creator]
max_turns: 60
timeout_seconds: 1800
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, Agent, SendMessage, ListAgents]
---

/ocd add CSV and JSON export to the reports module - GET /reports/:id/export?format=csv|json, plus a CLI command `reports export <id> --format csv`. reuse the existing report query, don't duplicate it
