# ECC Project Template

A portable project template for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) powered by the [Everything Claude Code](https://github.com/affaan-m/everything-claude-code) plugin. It includes pre-configured agents, skills, rules, and hooks so every new project starts with a battle-tested foundation.

## The Problem

Claude Code sessions are stateless. Every time you close a conversation, the context is gone — decisions you made, approaches you tried, what you planned to do next. For anything beyond a single-session task, this means:

- **Repeated explanations** — re-describing your project every time you start a new session
- **Forgotten decisions** — re-debating design choices you already settled
- **Wasted effort** — re-attempting approaches that already failed
- **Lost momentum** — no clear "pick up where I left off" path for multi-day projects

## The Solution: `/handoff` and the Roadmap System

This template ships two systems that give Claude Code **persistent project memory**.

### `/handoff` — Session Continuity

Run `/handoff` at the end of any session. Behind the scenes, an 8-agent fleet extracts everything that matters from your conversation:

- **What changed** — files modified, features added, bugs fixed
- **What was decided** — design choices and their rationale
- **What failed** — approaches that didn't work, so you never retry them
- **What's next** — a concrete "Continue" block with exact instructions for resuming

All of this is written to a compact session file (`docs/sessions/YYYY-MM-DD-descriptor.md`) and committed to git. The next time you open Claude Code, just say **"continue"** — the agent reads your roadmap and latest session, then picks up exactly where you left off.

### Roadmap — Structured Project Management

`/start-new-project` walks you through defining your project's phases, goals, and tasks, then generates a `docs/roadmap.json` that serves as the **single source of truth**. It tracks:

- **Phases and goals** with status progression (`pending` → `active` → `done`)
- **Task priorities** (high/medium/low) with a `next` pointer for current work
- **Blockers and parked ideas** so nothing falls through the cracks
- **Progress history** accumulated across every session

The roadmap updates automatically each time you run `/handoff`, keeping it in sync with actual progress.

### Why This Matters

| Without this template | With this template |
|-|-|
| Context lost every session | Decisions, failures, and plans persist in git |
| Manual project tracking | Roadmap updates automatically from your work |
| "Where was I?" every morning | Say "continue" and resume in seconds |
| Same mistakes repeated | Failed approaches recorded and avoided |
| Context window bloated with backstory | Compact session files (~60-100 lines) |

## Prerequisites

| Required | Optional |
|----------|----------|
| [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) | [Bun](https://bun.sh) |
| [Node.js](https://nodejs.org) (used by hooks) | Python 3 |
| [Git](https://git-scm.com) | [Prettier](https://prettier.io) |
| | [tmux](https://github.com/tmux/tmux) |

## Getting Started

1. Install the ECC plugin (one-time):

   ```sh
   claude plugin marketplace add https://github.com/affaan-m/everything-claude-code
   ```

2. Open this project in Claude Code.

3. Run `/start-new-project` to initialize your roadmap, goals, and first session.

## Configuration

MCP server configs live in `mcp-configs/mcp-servers.json`. Replace the placeholder API keys with your own. Every server listed is optional — configure only the ones you need.

## Resuming Work

Say **"continue"** in Claude Code. The agent will read `docs/roadmap.json` and your latest session files, then ask which workstream to pick up.

## Platform Note

Shell hooks (`.sh` scripts in `skills/` and the hook system) target macOS and Linux. Windows users should run this project inside WSL.
