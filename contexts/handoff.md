---
name: handoff
description: Context for handoff-focused sessions. Auto-loads roadmap state and guides continuity workflow.
---

# Handoff Context

This context activates when working on session handoffs and roadmap management.

## Key Files

- `docs/roadmap.json` - Source of truth (read first)
- `docs/roadmap.md` - Human-readable roadmap (auto-generated, do not edit)
- `docs/sessions/` - Session handoff files
- `skills/handoff/SKILL.md` - Handoff workflow instructions
- `skills/handoff/config.json` - Handoff configuration

## Workflow

1. Read `docs/roadmap.json` to understand current state
2. Check the `next` pointer for the current task
3. Review latest session file for continuity
4. Follow the handoff skill instructions for creating session files

## Task ID Format

`P{phase}.G{goal}.T{task}` - e.g., `P2.G1.T3`

## Commands

- `/handoff` - Create session handoff
- `/handoff --init` - Initialize roadmap
