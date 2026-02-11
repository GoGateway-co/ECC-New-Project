<!-- Mirror of skills/handoff/SKILL.md - keep in sync -->

---
name: handoff
description: Create session handoff files and update roadmap for project continuity across Claude Code sessions. Preserves context, decisions, and next steps.
---

# Handoff Skill (Thin Wrapper)

This skill creates session handoff files and maintains a project roadmap for continuity between Claude Code sessions.

## Architecture

This SKILL.md is a **thin wrapper** that:
1. Generates a compact session digest from conversation memory (Phase 1)
2. Delegates all orchestration to the `handoff-orchestrator` agent (Phases 0, 2, 3, 4)
3. Displays results and cleans up temp files

This reduces main context consumption from ~9,500 tokens to ~1,400 tokens (85% reduction).

## Flags

Parse these from the user's prompt text:

| Flag | Effect |
|------|--------|
| `--init` | Force init mode |
| `--task <id>` | Skip detection, use specified task |
| `--quick` | Abbreviated session file (no cumulative state) |
| `--no-commit` | Skip git operations |
| `--dry-run` | Show what would change without writing |
| `--review` | Enable two-pass mode with user review before Phase 4 |

## Mode Detection

Determine the mode based on current state:

| Condition | Mode |
|-----------|------|
| User passes `--init` flag | Init Mode |
| `docs/roadmap.json` does not exist | Init Mode |
| Roadmap exists with active tasks | Normal Mode |
| Roadmap exists but all tasks done | Empty State Mode |

---

## Init Mode

**Trigger**: `--init` flag or no `docs/roadmap.json` exists.

### Guard

If `docs/roadmap.json` already exists:

1. Read and display the current roadmap summary (name, phases, progress count)
2. Ask the user to confirm overwrite
3. If confirmed, create backup at `docs/roadmap.json.bak` before proceeding
4. If not confirmed, switch to Normal Mode

### Steps

1. **Delegate to handoff-init agent** for discovery and Q&A
   - Use Task tool with subagent_type="Bash"
   - Prompt: "You are the handoff-init agent. Follow the instructions in agents/handoff-init.md to initialize the project roadmap."
   - The agent scans project files and asks 6 questions about purpose, state, milestones, goals, priorities
   - Agent returns structured JSON with phases, goals, tasks

2. **Create roadmap.json** using Write tool:
   - Follow the schema at `schemas/roadmap.schema.json`
   - Set `version: "1.0"`
   - Set `origin.method: "handoff-init"`
   - Set `next` to first pending task
   - Add initial progress entry

3. **Create roadmap.md** using Write tool:
   - Generate from roadmap.json following roadmap-renderer.js logic
   - Include `<!-- Auto-generated from roadmap.json. Do not edit manually. -->` header

4. **Create first session file**:
   - Path: `docs/sessions/YYYY-MM-DD-[firstTaskId]-v1.md`
   - Content: initialization summary with Continue block

5. **Git commit** (unless `--no-commit`):
   ```
   git add docs/roadmap.json docs/roadmap.md docs/sessions/YYYY-MM-DD-*.md
   git commit -m "docs: handoff - roadmap initialized"
   ```

6. **Display** continuation prompt for next session

---

## Normal Mode

**Trigger**: `docs/roadmap.json` exists with active/pending tasks.

### Phase 1: Generate Session Digest (Main Context Only)

This is the **only** heavy work done in the main context. Synthesize a compact structured digest from conversation memory.

Generate the digest using this template:

```
SESSION_DIGEST
date: [YYYY-MM-DD - today's date]
task_hint: [scan conversation for P{n}.G{n}.T{n} mentions, or "none" for ad-hoc]
title: [Brief session title - use as markdown H1, 5-10 words]
status_change: [pending -> in_progress | in_progress -> done | etc.]
summary: [3-5 sentence summary of what happened this session]
decisions:
- "[Decision text]" | "[Rationale - explain WHY not just WHAT]"
- "[Another decision]" | "[Another rationale]"
failed_approaches:
- "[What was tried]" | "[Why it failed]"
- "[Another approach]" | "[Why it didn't work]"
files_modified:
- [path/to/file] | [Brief description of change]
- [another/file.js] | [What was changed]
next_action: [Single concrete next step - be specific]
known_breakage:
- [Description of known issues - OPTIONAL, omit if none]
```

**Instructions for generating the digest:**

1. **date**: Today's date in YYYY-MM-DD format
2. **task_hint**:
   - Scan conversation for mentions of task IDs (P1.G1.T1 format)
   - If found, use that task ID
   - If none found, set to "none"
3. **title**: Concise session title (5-10 words), e.g., "OTP Verification Implementation", "File Organization Terminal Awareness"
4. **status_change**: Based on what happened:
   - `pending -> in_progress` if this is the first session on a task
   - `in_progress -> done` if the task was completed
   - `in_progress -> in_progress` if work continues
   - `done -> done` if task was already complete
5. **summary**: 3-5 sentences covering:
   - What was the goal?
   - What was accomplished?
   - What's the current state?
6. **decisions**: List explicit choices made this session
   - Format: "Decision" | "Rationale"
   - Focus on WHY, not just WHAT
   - Include architectural choices, library selections, patterns adopted, things deliberately excluded
7. **failed_approaches**: Things that were tried but didn't work
   - Format: "What was tried" | "Why it failed"
   - Helps future sessions avoid dead ends
8. **files_modified**: Changed files with brief descriptions
   - Format: path/to/file | Brief change description
9. **next_action**: Single concrete next step for the next session
10. **known_breakage**: Optional - only if there are known issues

### Write Digest to Temp File

Write the generated digest to `.handoff-digest.tmp` in the project root.

### Launch Orchestrator Agent

Use the Task tool to launch the handoff-orchestrator agent:

```
You are the handoff orchestrator. Read the session digest from .handoff-digest.tmp
and execute Phases 0-4: task detection, 8-agent fleet, review/validation, assembly, file writing, git commit.

Configuration: skills/handoff/config.json
Project root: [current working directory]
Flags: [list all parsed flags, e.g., "--task P1.G1.T1 --no-commit"]

Follow the instructions in agents/handoff-orchestrator.md.

Return the HANDOFF_RESULT structured output when complete.
```

Use:
- `subagent_type: "general-purpose"`
- `model: "sonnet"` (orchestration requires reasoning)
- `description: "Orchestrate handoff workflow"`

### Handle Orchestrator Result

The orchestrator returns:

```
HANDOFF_RESULT
status: success|error
mode: dry-run|normal
session_file: docs/sessions/YYYY-MM-DD-descriptor-vN.md
roadmap_updated: true|false
git_committed: true|false
review_summary: [compact summary]
continue_block: [full ## Continue block verbatim]
error_message: [if status=error]
warnings: [any non-fatal issues]
```

**If status is "error"**: Display error_message and suggest retry or manual intervention.

**If status is "success"**:
1. Display the `continue_block` to the user
2. If there are warnings, display them
3. If `--dry-run` was used, display what would have been written

### Cleanup

Delete temp file:
```bash
rm -f .handoff-digest.tmp
```

---

## Empty State Mode

**Trigger**: Roadmap exists but all tasks are done (no pending/in_progress).

Present options:

```
All tasks in the roadmap are complete! What would you like to do?

1. **Quick-add**: Add new tasks to existing goals
2. **New phase**: Add a new phase with goals/tasks
3. **Reinit**: Start fresh with `--init` (backs up current)
4. **Archive**: Mark project as complete, create final session
5. **Capture-only**: Create session file without roadmap changes

Enter 1-5 or describe what you'd like to do:
```

Handle the user's choice appropriately (may require follow-up prompts).

---

## Error Handling

| Scenario | Action |
|----------|--------|
| roadmap.json doesn't exist | Offer `--init` |
| roadmap.json invalid JSON | Report error, suggest manual fix |
| roadmap.json fails validation | Report specific errors |
| docs/sessions/ doesn't exist | Create it (orchestrator handles this) |
| Orchestrator agent fails | Display error_message from HANDOFF_RESULT, suggest retry |
| Git operations fail | Orchestrator reports in warnings, session file still created |
| Non-git project | Orchestrator skips git operations |

---

## Implementation Notes

- **Token budget**: This wrapper consumes ~800 tokens when loaded, vs ~4,500 for the old SKILL.md
- **Orchestrator cost**: Runs in a fresh context with full token budget, reducing compaction risk
- **Temp file cleanup**: Always delete `.handoff-digest.tmp` after orchestrator completes (success or failure)
- **Review mode**: `--review` flag enables two-pass mode where orchestrator returns summary for user approval before Phase 4
