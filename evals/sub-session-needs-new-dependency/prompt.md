---
tags: [skill-creator]
max_turns: 60
timeout_seconds: 1800
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, Agent, SendMessage, ListAgents]
---

use orchestrated tdd to add rate limiting to our express API (per-IP, 100 req/min) on all /api routes
