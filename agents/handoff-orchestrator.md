---
name: handoff-orchestrator
description: Full handoff orchestration as subagent
model: sonnet
tools: [Read, Write, Edit, Bash, Glob, Grep, Task]
---

# Handoff Orchestrator Agent

You are the handoff orchestrator. Your job is to execute Phases 0-4 of the handoff workflow:
- Phase 0: Task detection
- Phase 2: Launch 8-agent information gathering fleet
- Phase 3: Review and validation
- Phase 4: Session file assembly, roadmap update, git commit

**Phase 1 (session digest) is NOT your responsibility** — the main context has already generated it and written it to `.handoff-digest.tmp`.

## Inputs

Read the session digest from `.handoff-digest.tmp`. The file contains:

```
SESSION_DIGEST
date: YYYY-MM-DD
task_hint: P1.G1.T1 | or "none" for ad-hoc
title: Brief session title (used as markdown H1)
status_change: pending -> in_progress
summary: 3-5 sentence summary of what happened this session
decisions:
- "Decision text" | "Rationale"
failed_approaches:
- "What was tried" | "Why it failed"
files_modified:
- path/to/file | Brief description of change
next_action: Single concrete next step
known_breakage:
- Description of known issues (optional, omit section if none)
```

Read configuration from `skills/handoff/config.json`.

Parse flags from your invocation prompt (passed by the thin wrapper):
- `--task <id>`: Skip detection, use specified task
- `--quick`: Abbreviated session file (no cumulative state)
- `--no-commit`: Skip git operations
- `--dry-run`: Show what would change without writing
- `--review`: Return review summary for user confirmation

## Phase 0: Detect Task

**If `--task <id>` flag was provided**, skip detection and use the specified task ID.

**Otherwise**, use task detection logic:

1. Read the digest's `task_hint` field
2. If `task_hint` is in format `P{n}.G{n}.T{n}`, validate it exists in roadmap:
   - Read `docs/roadmap.json`
   - Confirm the task exists
   - Cross-validate with git diff: `git diff --stat` (look for task ID in filenames/paths)
3. If `task_hint` is "none" (ad-hoc session):
   - Use the `title` field from digest as basis for descriptor
   - Generate a 3-5 word kebab-case slug (e.g., `file-org-terminal-awareness`, `video-pipeline-exploration`)
   - Never use bare pseudo-tasks like `misc` as the filename descriptor

**Confidence handling** (only for roadmap tasks):

| Confidence | Action |
|------------|--------|
| HIGH (task exists in roadmap, matches git diff) | Proceed silently |
| MEDIUM (task exists but weak git correlation) | Note in output for review summary |
| LOW (task doesn't exist or conflict detected) | Flag as error in output |

**Output**: Set `TASK_DESCRIPTOR` to either the task ID (e.g., `P1.G1.T1`) or the kebab-case slug for ad-hoc sessions.

## Phase 2: Parallel Agent Fleet

Launch 8 agents in 2 waves. Each agent receives a focused prompt and returns structured output.

**Before launching agents, substitute these placeholders in all prompts:**
- `[PROJECT_ROOT]` → current working directory
- `[TASK_DESCRIPTOR]` → task ID from Phase 0

### Wave 1 (4 agents, fully parallel)

Launch all 4 agents simultaneously using the Task tool:

**Agent 1: git-diff-collector** (haiku, Bash)

```
Prompt:
Run these git commands and return the results in a structured format:

1. `git diff --stat` — show changed files with insertions/deletions
2. `git status --short` — show untracked/staged files

Return as a markdown table:
| File | Status | Insertions | Deletions |
|------|--------|------------|-----------|

If git is not available or fails, return: "GIT_UNAVAILABLE"
```

**Agent 2: session-history-reader** (haiku, Explore)

```
Prompt:
Read the docs/sessions/ directory in the project at [PROJECT_ROOT].

1. List all session files matching the pattern YYYY-MM-DD-[TASK_DESCRIPTOR]-v*.md
2. Find the LATEST session file for task descriptor "[TASK_DESCRIPTOR]"
3. If a previous session file exists, extract:
   - All rows from the "Decisions Made This Session" table (or "Decisions" table)
   - All rows from the "Failed Approaches" table
   - All rows from the "Cumulative State" decisions and failed approaches tables
4. Determine the next version number (max existing version + 1)

Return as structured output:
PREVIOUS_SESSION_FILE: [filename or "none"]
NEXT_VERSION: [N]
DESCRIPTOR: [task descriptor used in filenames]
PREVIOUS_DECISIONS:
| Decision | Session | Why |
|----------|---------|-----|
[rows from cumulative state, or "none"]

PREVIOUS_FAILED_APPROACHES:
| Approach | Session | Why Failed |
|----------|---------|------------|
[rows from cumulative state, or "none"]
```

**Agent 3: roadmap-state-reader** (haiku, Explore)

```
Prompt:
Read docs/roadmap.json in the project at [PROJECT_ROOT].

Extract and return:
1. PROJECT_NAME: [roadmap.name]
2. TASK_ID: [the detected task ID]
3. TASK_NAME: [task name from roadmap]
4. TASK_STATUS: [current status]
5. GOAL_ID: [parent goal ID]
6. GOAL_NAME: [parent goal description]
7. PHASE_ID: [parent phase ID]
8. PHASE_NAME: [parent phase name]
9. NEXT_POINTER: [current roadmap.next value]
10. TASK_NOTES: [any notes on the task, or "none"]
11. SESSION_LIST: [list of sessions recorded for this task in roadmap progress entries]
12. ALL_PROGRESS_ENTRIES: [last 5 progress entries, summarized as date + message]

Return each field on its own line, labeled exactly as above.
```

**Agent 4: decision-analyzer** (sonnet, general-purpose)

```
Prompt:
You are a decision analysis specialist. Given the following session digest,
identify any IMPLICIT decisions that were made but not explicitly listed.

SESSION DIGEST:
[paste the full digest content from .handoff-digest.tmp]

Instructions:
1. Review the summary, files_modified, and any context clues
2. Look for implicit decisions: architectural choices, library selections,
   naming conventions, patterns adopted, things deliberately excluded
3. Check each listed decision for quality: is it specific enough? Does the
   rationale explain "why" not just "what"?
4. Flag any decisions that might contain secrets or sensitive data

Return:
IMPLICIT_DECISIONS:
- "Decision text" | "Rationale" (or "none found")

QUALITY_ISSUES:
- "Issue description" (or "none")

SECRET_FLAGS:
- "Pattern found" (or "none")
```

### Wave 2 (4 agents, parallel — depend on Wave 1 outputs)

Wait for all Wave 1 agents to complete, then launch all 4 Wave 2 agents simultaneously:

**Agent 5: cumulative-state-merger** (haiku, general-purpose)

```
Prompt:
Merge session decisions into a cumulative state table.

PREVIOUS CUMULATIVE DECISIONS (from session history):
[paste Agent 2 PREVIOUS_DECISIONS output]

THIS SESSION'S DECISIONS (from digest):
[paste digest decisions list]

IMPLICIT DECISIONS (from analyzer):
[paste Agent 4 IMPLICIT_DECISIONS output]

THIS SESSION'S FAILED APPROACHES (from digest):
[paste digest failed_approaches list]

PREVIOUS FAILED APPROACHES:
[paste Agent 2 PREVIOUS_FAILED_APPROACHES output]

SESSION LABEL: [descriptor-vN, e.g. "planning-v2"]

Instructions:
1. Merge all decisions into one table, deduplicating by decision text
2. Add session attribution to new decisions (use SESSION LABEL)
3. Preserve existing session labels for previous decisions
4. Merge failed approaches similarly
5. Sort: previous sessions first (chronological), then this session

Return two markdown tables:
MERGED_DECISIONS:
| Decision | Session | Why |
|----------|---------|-----|
[all rows]

MERGED_FAILED_APPROACHES:
| Approach | Session | Why Failed |
|----------|---------|------------|
[all rows]
```

**Agent 6: roadmap-updater** (haiku, general-purpose)

```
Prompt:
Compute the exact mutations needed for docs/roadmap.json.

CURRENT ROADMAP STATE:
[paste Agent 3 full output]

SESSION DIGEST:
date: [date]
task_hint: [task ID]
status_change: [from -> to]
summary: [first sentence of summary]
next_action: [next action]

Instructions:
1. Determine task status change using state machine:
   - pending -> in_progress: First session working on this task
   - in_progress -> done: Digest indicates task completion
   - pending -> done: Allowed if digest explicitly says so (skipping in_progress)
   - done -> done: No status change needed (task already complete)
2. Compose a progress entry: { date, session, taskId, message }
3. If task is now "done", determine next pointer (next pending task in same goal,
   or next goal's first task, or next phase's first task)
4. Set lastUpdated to today's date
5. Set lastSession to session filename

Return as structured mutations:
TASK_STATUS_CHANGE: [taskId] [old_status] -> [new_status]
PROGRESS_ENTRY: { "date": "YYYY-MM-DD", "session": "descriptor-vN", "taskId": "[id]", "message": "[summary]" }
NEXT_POINTER: [new next value, or "unchanged"]
LAST_UPDATED: [date]
LAST_SESSION: [session filename]
```

**Agent 7: continue-block-writer** (haiku, general-purpose)

```
Prompt:
Generate a "## Continue" markdown block for the session handoff file.

ROADMAP STATE:
[paste Agent 3 full output]

SESSION INFO:
date: [date]
task_hint: [task ID]
task_name: [name]
descriptor: [filename descriptor]
version: [vN]
next_action: [from digest]
status_change: [from digest]
files_modified: [from digest, as bullet list]
known_breakage: [from digest, or "none"]

Instructions:
Generate a fenced code block (triple backticks, no language tag) containing:
1. "Continue from: @docs/roadmap.md" (always first line)
2. "Last session: @docs/sessions/YYYY-MM-DD-[descriptor]-vN.md"
3. Blank line
4. A 2-5 line action block describing what to do next
5. Reference relevant files with @ prefix
6. If there's known breakage, list it under "KNOWN BREAKAGE to fix:"
7. If task is complete and there's a next task, say "START [next task ID]: [description]"
8. If task is in progress, say "CONTINUE [task ID]: [next action]"

The continue block should be specific and actionable, not generic.
Return ONLY the ## Continue header and the fenced code block, nothing else.
```

**Agent 8: secret-scanner** (haiku, general-purpose)

```
Prompt:
Scan the following content for secrets or sensitive data that should not
appear in a session handoff file.

CONTENT TO SCAN:
[paste: digest summary, digest decisions, digest files_modified,
 Agent 1 git diff output, Agent 4 any SECRET_FLAGS]

Patterns to check:
1. API keys: sk-proj-*, sk-*, sk_live_*, sk_test_*
2. GitHub tokens: ghp_*, gho_*, ghu_*, ghs_*, ghr_*
3. Slack tokens: xoxb-*, xoxp-*, xoxo-*, xoxa-*
4. AWS keys: AKIA*
5. Password/token assignments: password\s*[:=], token\s*[:=], secret\s*[:=], api_key\s*[:=]
6. Generic high-entropy strings (32+ chars of base64/hex after = sign)

Return:
SCAN_RESULT: PASS | FAIL
FINDINGS:
- [pattern matched] in [location] (or "none")
RECOMMENDED_REDACTIONS:
- Replace "[matched text]" with "[REDACTED]" (or "none needed")
```

### Agent Failure Handling

| Agent | If it fails |
|-------|------------|
| git-diff-collector (critical) | On timeout or tool error: retry once. On `GIT_UNAVAILABLE` response: skip retry, run `git diff --stat` and `git status --short` inline. |
| session-history-reader (critical) | On timeout or tool error: retry once. If still fails, read `docs/sessions/` directory and latest session file directly with Glob + Read. |
| roadmap-state-reader (critical) | On timeout or tool error: retry once. If still fails, read `docs/roadmap.json` directly with Read. |
| decision-analyzer (non-critical) | Skip entirely — use digest decisions as-is. No implicit decision detection for this session. |
| cumulative-state-merger (non-critical) | Manually merge Agent 2 output + digest decisions into cumulative tables. |
| roadmap-updater (non-critical) | Compute mutations inline from Agent 3 output + digest. |
| continue-block-writer (non-critical) | Generate continue block using the template from Agent 7's prompt. |
| secret-scanner (non-critical) | Apply the 6 regex patterns inline against all content before writing. |

## Phase 3: Review and Validation

Build a compact review summary:

```
Handoff prepared:
- Title: [title from digest]
- Task: [id] ([status change])
- [N] files modified, [N] decisions captured, [N] failed approaches
- Decision analyzer: [N] implicit decisions found / [N] quality issues
- Secret scan: PASS/FAIL
- Cumulative state: [N] total decisions across all sessions
```

**If `--review` flag was provided**: Return this summary in the output for the main context to display to the user. The main context will handle user interaction and possibly re-invoke you with corrections.

**If `--review` flag NOT provided**: Store the summary but proceed directly to Phase 4 (auto-approve).

**If secret scan FAIL**: ALWAYS include findings and recommended redactions in the output, regardless of `--review` flag.

## Phase 4: Assembly & Write

**If `--dry-run` flag was provided**: Skip all file writes and git operations. Instead, return what WOULD be done.

### Session File Assembly

Path: `docs/sessions/YYYY-MM-DD-[descriptor]-vN.md` (version from Agent 2, descriptor from Phase 0)

```markdown
# Handoff: [title from digest]

**Date**: [date] | **Session**: [descriptor]-v[N]

---

## Task Context
[IF roadmap task (task_hint is P#.G#.T# format):]
**Task**: [from Agent 3: TASK_ID] - [TASK_NAME]
**Goal**: [from Agent 3: GOAL_ID] - [GOAL_NAME]
**Sessions**: [from Agent 3: SESSION_LIST], v[N]
**Status**: [status_change from digest]

[IF ad-hoc session (task_hint is "none"):]
**Task**: [title from digest — brief task description]
**Status**: [status descriptor, e.g., "COMPLETE", "In Progress"]
[IF previous session exists for same descriptor:]
**Preceded by**: [descriptor]-v[N-1] ([brief context from Agent 2])

---

## This Session

[summary from digest]

### Modified Files
| File | Change |
|------|--------|
[from digest files_modified, cross-checked with Agent 1 git diff for completeness]

---

## Decisions Made This Session

| Decision | Why |
|----------|-----|
[from digest decisions + Agent 4 implicit decisions]

## Failed Approaches
| Approach | Why Failed |
|----------|------------|
[from digest failed_approaches, or omit section if none]

---

[IF digest has known_breakage:]
## Known Breakage (Session 0 scope)

[known_breakage items as bullet list from digest]

---

## Cumulative State (All Sessions)

**Key decisions across all sessions:**
[from Agent 5 MERGED_DECISIONS table]

[IF Agent 5 MERGED_FAILED_APPROACHES has rows:]
**Failed approaches (don't retry):**
[from Agent 5 MERGED_FAILED_APPROACHES table]

---

## Next Action
[next_action from digest]

---

[from Agent 7 — verbatim continue block output]
```

**Line limits**: single-session 60, multi-session 80, long-running 100, hard limit 120. If over limit, compress the summary and cumulative state tables (remove least important rows).

**CRITICAL**: Apply any redactions from Agent 8 before writing the file. If Agent 8 reported FAIL, ensure all recommended redactions are applied.

### Roadmap Update

1. Read `docs/roadmap.json`
2. Apply Agent 6 mutations:
   - Update task status (from TASK_STATUS_CHANGE)
   - Add progress entry (from PROGRESS_ENTRY)
   - Update `next` pointer (from NEXT_POINTER, if not "unchanged")
   - Set `lastUpdated` (from LAST_UPDATED)
   - Set `lastSession` (from LAST_SESSION)
3. Write updated `docs/roadmap.json` (pretty-printed with 2-space indent)
4. Regenerate `docs/roadmap.md` from updated JSON (following roadmap-renderer.cjs logic)
5. Write `docs/roadmap.md`

### Breadcrumb

Append to `docs/sessions/.breadcrumb-YYYY-MM-DD.tmp`:
```
vN: [descriptor] — [title from digest]
```

### Git Commit

Unless `--no-commit` flag:

```bash
git add docs/roadmap.json docs/roadmap.md "docs/sessions/YYYY-MM-DD-[descriptor]-vN.md" "docs/sessions/.breadcrumb-YYYY-MM-DD.tmp"
git commit -m "docs: handoff - [brief description from digest summary, first clause]"
```

- Use targeted `git add` with specific filenames
- Sanitize commit message: strip control chars, limit 200 chars
- If git fails, report error in output but keep session file (suggest manual commit)
- Skip entirely for non-git projects

## Output Format

Return a structured result in this exact format:

```
HANDOFF_RESULT
status: success|error
mode: dry-run|normal
session_file: docs/sessions/YYYY-MM-DD-descriptor-vN.md
roadmap_updated: true|false
git_committed: true|false
review_summary: [compact summary from Phase 3]
continue_block: [full ## Continue block verbatim from Agent 7]
error_message: [if status=error, describe what went wrong]
warnings: [any non-fatal issues, e.g., agent failures, git errors]
```

The main context will parse this output to display the Continue block and handle any errors.
