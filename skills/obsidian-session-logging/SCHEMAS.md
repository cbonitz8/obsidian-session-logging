# Frontmatter Schemas

Every file carries standardized YAML frontmatter. These are the **schema** — tooling and Dataview queries depend on them. Always include all required fields for the file's `type`.

## Sync fields (optional — only with a sync plugin like Snobby)

- `sn_category` — maps to remote document category (see Category reference)
- `sn_project` — maps to remote project field
- `sn_tags` — comma-separated tags (e.g., "archived", "complete")
- `sn_synced: false` — set on creation/edit; the sync plugin flips it to true after push

### Category reference

The `sn_category` values below are built-in defaults; new ones may be added in ServiceNow at any time. To discover all available categories, read the sync plugin's settings:

```bash
cat "<vault>/.obsidian/plugins/snobby/data.json" | grep -A 50 '"categories"'
```

The `categories` object maps SN values (e.g., `story_time`) to folder names (e.g., `"Story Time"`). Use the SN value for `sn_category`. If a needed category isn't mapped, the plugin auto-derives the folder name from the value.

## Session Log

```yaml
---
project: "Project Name"         # required — must match overview's project name exactly
date: YYYY-MM-DD                # required
type: session                   # required — literal "session"
author: <identifier>            # required — consistent per person (e.g., "alice", "bob")
status: in-progress|complete    # required — set to "complete" at wrap-up
plan: "[[path/to/plan]]"        # optional — link to plan being executed
tags: []                        # optional — action tags (see REFERENCE.md → Tags)
records:                        # optional — instance records touched
  - table: table_name
    sys_id: abc123
    name: RecordName
    action: modified|created|deleted
sn_category: session_log
sn_project: "Project Name"
sn_tags: ""
sn_synced: false
---
```

- Append after each significant change
- Tag post-deploy actions with `#post-deploy` immediately
- Include changed files at end of session
- Set `status: complete` during the wrap-up checklist
- Body sections: `## Summary`, `## Changes`, `## Open Issues`, `## Changed Files`, `## Resume pointer`, `## Session Notes`. The **Resume pointer** is the post-`/clear` handoff — machine-crisp state for a fresh session to continue from: current update set/branch/working state, what landed and where, the single NEXT action, the frontier, and any pending verification. Add this section to the vault's `Templates/Session Log` so every new log carries it.

## Daily Log

```yaml
---
date: YYYY-MM-DD                # required
type: daily-log                 # required — literal "daily-log"
author: <identifier>            # required
projects:                       # required — list of projects touched
  - "Project Name"
sn_category: daily_log
sn_project: ""
sn_tags: ""
sn_synced: false
---
```

- **Sessions**: link to all session logs from the day
- **Projects Active**: heading per project linking to its overview, brief current state
- **Open Threads / Next Steps**: what the next session picks up
- A daily log is a **context compass** — for implementation detail, read the overview and code

## Design Spec (Plan)

```yaml
---
project: "Project Name"         # required
type: plan                      # required — literal "plan"
status: active|completed|superseded  # required
date: YYYY-MM-DD                # required — creation date
author: <identifier>            # required
qa: true                        # optional — set on QA testing guides/checklists
superseded_by: "[[link]]"       # required if status: superseded
sn_category: design_spec
sn_project: "Project Name"
sn_tags: ""
sn_synced: false
---
```

- The overview links to the active plan; session logs reference plans via `plan:` frontmatter
- When complete: `status: completed`. When replaced: `status: superseded` + `superseded_by:` link
- **QA docs**: testing guides (not implementation tasks) set `qa: true` — this excludes their checkboxes from the dashboard's Open Tasks and Post-Deploy Actions queries

## Project Overview

```yaml
---
project: "Project Name"         # required — canonical name, used for matching
status: active                  # required — active|complete|on-hold
type: project                   # required — literal "project" or "area"
sn_category: project_overview
sn_project: "Project Name"
sn_tags: ""
sn_synced: false
---
```

**`type: area`** — ongoing responsibilities with no end date (e.g., "Platform Maintenance"). Same folder structure as projects (overview + Session Logs/ + Design Specs/) but never "complete" — status stays `active`. Logs and standups reference areas exactly like projects.

**Overview content — what belongs:**
- **What it is** — what the project/area does, who uses it, what problem it solves
- **How it works** — high-level user-facing workflow (not implementation internals)
- **Ecosystem** — how it connects to other projects, with `[[wikilinks]]`
- **Current status** — one scannable line summarizing where things stand

**Overview content — what does NOT belong:** field-level data-model tables, REST endpoint lists, business-rule firing orders, code snippets, phase-by-phase build history, plans tables (plans are discoverable via backlinks and session logs).

**Technical detail** goes in a separate reference doc at `<project>/Design Specs/<project-name>-technical-reference.md` (`type: plan`, `status: active`), linked from the overview's Ecosystem section. Only create one for projects with substantial technical detail (data models, APIs, architecture).

## Standup

```yaml
---
date: YYYY-MM-DD                # required — the meeting day, NOT the work day
type: standup                   # required — literal "standup"
sn_category: standup
sn_project: ""
sn_tags: ""
sn_synced: false
---
```

- One file per **standup meeting day**, named by that date: `Standups/YYYY-MM-DD.md`
- Shared by all team members — each person adds their own `### <author>` section
- No `author` field in frontmatter (multiple authors per file)
- Dashboard links to the latest standup via dataview query

The date arithmetic and the per-author section rules live with the wrap-up step that writes standups (SKILL.md → Wrapping Up → Standup notes), since that is the only place they fire.

## Component Doc

```yaml
---
type: component                 # required — literal "component"
name: "Component Name"          # required — display name
sn_category: reference
sn_project: ""
sn_tags: ""
sn_synced: false
---
```

## Index

```yaml
---
type: index                     # required — literal "index"
sn_category: index
sn_project: ""
sn_tags: ""
sn_synced: false
---
```

- Single file: `Index/index.md`, auto-maintained by the LLM — not manually edited
- Format and update workflows live in SKILL.md → Vault Index
