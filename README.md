# ECC Project Template

A portable project template for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) powered by the [Everything Claude Code](https://github.com/affaan-m/everything-claude-code) plugin. It includes pre-configured agents, skills, rules, and hooks so every new project starts with a battle-tested foundation.

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
