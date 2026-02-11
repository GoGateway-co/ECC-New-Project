---
description: Initialize a new project from this ECC template. Creates roadmap, goals, and first session.
---

# /start-new-project

Initialize a new project from this ECC template.

## Steps

1. **Ask** the user for:
   - Project name
   - One-sentence purpose/description
   - Key goals (2-5 bullet points)

2. **Run** `/handoff --init` to create the initial roadmap and session structure:
   - Creates `docs/roadmap.json` with the project name, description, and goals
   - Creates `docs/roadmap.md` (human-readable view)
   - Creates the first session file in `docs/sessions/`

3. **Update** `CLAUDE.md` to replace the template title with the actual project name.

4. **Confirm** to the user that the project is ready, showing:
   - The roadmap file location
   - The first session file location
   - How to resume work with `/continue`
