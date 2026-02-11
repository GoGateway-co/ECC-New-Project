---
description: Pull latest ECC updates from the global plugin into this project.
---

# /sync-ecc

Pull the latest Everything Claude Code (ECC) updates from the global marketplace plugin into this project.

## Steps

1. **Read versions**:
   - Global: `~/.claude/plugins/marketplaces/everything-claude-code/.claude-plugin/plugin.json` → `version`
   - Project: `.claude-plugin/plugin.json` → `version`

2. **Compare**: If versions match, report "Already up to date (vX.Y.Z)" and stop.

3. **Show changes**: Summarize what directories differ between global and project (agents, rules, skills, commands, hooks, contexts, schemas, plugins, mcp-configs, root configs).

4. **Sync** the following from global → project, replacing existing files:
   - `agents/` — all `.md` files (keep project-specific agents like `handoff-*.md`)
   - `rules/` — full replace with global
   - `skills/` — full replace with global (keep `handoff/`)
   - `commands/` — copy all global commands (keep project-specific: `start-new-project.md`, `sync-ecc.md`)
   - `hooks/hooks.json` — replace
   - `contexts/` — replace with global (keep `handoff.md`)
   - `schemas/` — replace with global (keep `roadmap.schema.json`)
   - `plugins/` — full replace
   - `mcp-configs/` — full replace
   - Root configs: `.markdownlint.json`, `commitlint.config.js`, `eslint.config.js`

5. **Update** `.claude-plugin/plugin.json` version to match global.

6. **Report** the updated version and list of synced directories.

## Preserved files (never overwritten)
- `CLAUDE.md`
- `.claude/settings.json`, `.claude/settings.local.json`
- `skills/handoff/`
- `agents/handoff-*.md`
- `contexts/handoff.md`
- `schemas/roadmap.schema.json`
- `commands/start-new-project.md`, `commands/sync-ecc.md`
- `docs/` (all project documentation)
