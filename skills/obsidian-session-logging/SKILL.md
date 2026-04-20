---
name: obsidian-session-logging
description: Use when starting a new conversation, wrapping up a work session, or when the user asks to update logs, generate standup notes, create session entries, or archive old files. Also use when beginning work on a project to read recent context.
---

# Obsidian Session Logging

Track work sessions, daily progress, and standup notes in an Obsidian vault.

## CLI Configuration

This skill uses the **Obsidian CLI** (bundled with Obsidian 1.12+) as the primary interface for vault operations.

- **CLI binary:** Detect by platform:
  - **macOS:** `/Applications/Obsidian.app/Contents/MacOS/obsidian`
  - **Windows:** `%LOCALAPPDATA%\Obsidian\Obsidian.exe`
  - **Linux:** `obsidian` (if on PATH), or check `/usr/bin/obsidian`, `/snap/bin/obsidian`
  - If none found, try `obsidian` as a bare command (works when on PATH)
- **Vault name:** Read from `CLAUDE.md` → `Obsidian Vault Integration` → `Vault name`
- **Vault path:** Read from `CLAUDE.md` → `Obsidian Vault Integration` → `Vault path`

All CLI commands use `vault="<name>"` (e.g., `vault="My Vault"`). If the CLI is unavailable (Obsidian not running, binary not found), fall back to Read/Write/Edit/Glob/Grep — see **Fallback Mode** at the end.

### CLI Command Reference

```bash
# Reading
obsidian vault="X" read path="<path>"
obsidian vault="X" property:read name="<prop>" path="<path>"
obsidian vault="X" properties path="<path>" format=yaml

# Creating files from templates
obsidian vault="X" create path="<path>" template="<Template Name>"
obsidian vault="X" property:set name="<prop>" value="<val>" path="<path>"
obsidian vault="X" property:set name="<prop>" value="<val>" type=list path="<path>"

# Appending / prepending content
obsidian vault="X" append path="<path>" content="<text>"
obsidian vault="X" prepend path="<path>" content="<text>"

# Finding files and content
obsidian vault="X" files folder="<folder>"
obsidian vault="X" search query="<text>" path="<folder>" format=json
obsidian vault="X" backlinks path="<path>"

# Task management
obsidian vault="X" tasks todo                          # all incomplete tasks
obsidian vault="X" tasks todo path="<folder>"          # scoped to folder
obsidian vault="X" tasks todo verbose                  # grouped by file with line numbers
obsidian vault="X" task done path="<path>" line=<n>    # check off a task

# Moving / deleting
obsidian vault="X" move path="<old>" to="<new>"
obsidian vault="X" delete path="<path>"
```

**Notes:**
- Quote values with spaces: `value="My Project"`
- Use `\n` for newlines in `content=` values
- `file=` resolves by name (like wikilinks); `path=` is exact relative path
- **Mid-file edits** still require Read + Edit tools (CLI has no edit/replace command)

## Vault Structure

Read vault path from the project's CLAUDE.md or memory. Expected structure:

```
<vault>/
  <project>/
    Project Overviews/
      <Project Name> Overview.md  # Source of truth for current state
    Session Logs/
      description.md
    Design Specs/
      plan-name.md
  Daily Logs/                     # Top-level, not per-project
    description.md
  Standups/                       # One file per day, shared by all authors
    YYYY-MM-DD.md
  Resources/                      # Reusable component docs
    Component Name.md
  Templates/                      # Obsidian templates
  Project Overviews/
    ✍️ Dashboard.md               # Rolling dashboard with dataview queries
```

**Key structure rules:**
- Projects are **top-level folders** — no `Projects/` prefix
- Daily Logs are **top-level** — no `Areas/` prefix
- Overviews live at `<project>/Project Overviews/<Project Name> Overview.md`
- Component docs live directly in `Resources/` — no `Components/` subfolder
- **No Archive folder** — archived docs stay in their project folder, marked via frontmatter (e.g., `tags: "archived"` or sync plugin tags)
- `Session Logs/` (not `Sessions/`) and `Design Specs/` (not `Plans/`)

## Naming Conventions

| File Type | Pattern | Example |
|-----------|---------|---------|
| Session log | `<brief description>.md` | `API refactor + auth flow.md` |
| Daily log | `<brief description>.md` | `API refactor + deployment.md` |
| Design spec | `<kebab-name>.md` | `auth-migration-plan.md` |
| Standup | `YYYY-MM-DD.md` | `2026-04-06.md` |
| Project overview | `<Project Name> Overview.md` | `My Project Overview.md` |
| Component doc | `<Display Name>.md` | `Data Grid Component.md` |

- **No dates in filenames** — dates are tracked in frontmatter (`date:` property). Old files with dates in the name are legacy; new files omit the date entirely.
- Session/daily descriptions: lowercase, concise, use `+` to join topics
- Plan names: kebab-case
- Overview files are named with the project prefix so they're distinguishable in flat views (search results, browser views)

## Templates

Templates live in the `Templates/` folder and are used via the CLI's `create template=` parameter. After creating a file from a template, use `property:set` to fill in dynamic values.

**Available templates:** Session Log, Daily Log, Design Spec, Standup, Project Overview, Component Doc

**Creation workflow:**
```bash
# 1. Create file from template ({{date}} resolves automatically)
obsidian vault="X" create path="<project>/Session Logs/<name>.md" template="Session Log"

# 2. IMPORTANT: Clear inherited sn_sys_id and set correct sn_category so this
#    file gets its own SN record and lands in the right folder on sync
obsidian vault="X" property:set name="sn_sys_id" value="" path="<project>/Session Logs/<name>.md"
obsidian vault="X" property:set name="sn_synced" value="false" path="<project>/Session Logs/<name>.md"
obsidian vault="X" property:set name="sn_category" value="session_log" path="<project>/Session Logs/<name>.md"

# 3. Set dynamic frontmatter properties
obsidian vault="X" property:set name="project" value="Project Name" path="<project>/Session Logs/<name>.md"
obsidian vault="X" property:set name="author" value="<author>" path="<project>/Session Logs/<name>.md"
obsidian vault="X" property:set name="sn_project" value="Project Name" path="<project>/Session Logs/<name>.md"

# 4. Append dynamic body content
obsidian vault="X" append path="<project>/Session Logs/<name>.md" content="## Session Notes\n\nStarting work on..."
```

**WARNING (if using sync plugin):** Templates may contain sync identifiers (e.g., `sn_sys_id`) and `sn_category: template`. The `create template=` command copies ALL frontmatter. You MUST clear `sn_sys_id` and set the correct `sn_category` immediately after creation, or the new file will collide with the template's remote record and land in the wrong folder on sync.

## Frontmatter Properties (Schema)

Every file has standardized YAML frontmatter. These properties are the **schema** — tooling and Dataview queries depend on them. Always include all required fields.

**Sync fields (optional — only if using a sync plugin like Snobby):**
- `sn_category` — maps to remote document category (see category reference below)
- `sn_project` — maps to remote project field
- `sn_tags` — comma-separated tags (e.g., "archived", "complete")
- `sn_synced: false` — set to false on creation/edit, sync plugin sets to true after push

**Category reference:** The `sn_category` values listed in the schemas below are the built-in defaults. New categories may be added in ServiceNow at any time. To discover all available categories and their values, read the sync plugin's settings file:
```bash
cat "<vault>/.obsidian/plugins/snobby/data.json" | grep -A 50 '"categories"'
```
The `categories` object maps SN values (e.g., `story_time`) to folder names (e.g., `"Story Time"`). Use the SN value for `sn_category`. If a category you need isn't in the mapping, the sync plugin will auto-derive the folder name from the value (e.g., `story_time` → "Story Time").

### Session Log
```yaml
---
project: "Project Name"         # required — must match overview's project name exactly
date: YYYY-MM-DD                # required
type: session                   # required — literal "session"
author: <identifier>            # required — consistent per person (e.g., "alice", "bob")
status: in-progress|complete    # required — set to "complete" at wrap-up
plan: "[[path/to/plan]]"        # optional — link to plan being executed
tags: []                        # optional — action tags (see Tags section)
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
- Set `status: complete` during wrap-up checklist

### Daily Log
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
- Daily logs are a **context compass** — for implementation details, read the overview and code

### Design Spec (Plan)
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
- The overview links to the active plan
- Session logs reference plans via `plan:` frontmatter
- When complete: set `status: completed`
- When replaced: set `status: superseded` + `superseded_by:` link
- **QA docs**: Plans that are testing guides for QA (not implementation tasks) must include `qa: true` in frontmatter. This excludes their checkboxes from the dashboard's Open Tasks and Post-Deploy Actions queries.

### Project Overview
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

**`type: area`** — Use for ongoing responsibilities that don't have an end date (e.g., "Platform Maintenance", data quality work, shared utility upkeep). Areas use the same folder structure as projects (overview + Session Logs/ + Design Specs/) but are never "complete" — their status stays `active`. Session logs, daily logs, and standups reference areas exactly like projects.

**Overview content — what belongs:**
- **What it is** — what the project/area does, who uses it, what problem it solves
- **How it works** — high-level user-facing workflow (not implementation internals)
- **Ecosystem** — how it connects to other projects, with `[[wikilinks]]`
- **Current status** — one scannable line summarizing where things stand

**Overview content — what does NOT belong:**
- Field-level data model tables
- REST API endpoint lists
- Business rule firing orders
- Code snippets or implementation details
- Phase-by-phase build history
- Plans tables (plans are discoverable via backlinks and session logs)

**Technical detail** goes in a separate reference doc at `<project>/Design Specs/<project-name>-technical-reference.md` (`type: plan`, `status: active`). Link to it from the overview's Ecosystem section. Only create reference docs for projects with substantial technical detail (data models, APIs, architecture).

### Standup
```yaml
---
date: YYYY-MM-DD                # required
type: standup                   # required — literal "standup"
sn_category: standup
sn_project: ""
sn_tags: ""
sn_synced: false
---
```
- One file per day, named by date: `Standups/YYYY-MM-DD.md`
- Shared by all team members — each person adds their section under the date
- No `author` field in frontmatter (multiple authors per file)
- Dashboard links to the latest standup via dataview query

### Component Doc
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

## Tags

Tags mark **action items** on task checkboxes. Keep the set small — every tag should power a query or workflow.

| Tag | Meaning | Use on |
|-----|---------|--------|
| `#post-deploy` | Requires action on the target instance after code deploy | Task checkboxes in session logs, plans |
| `#blocked` | Cannot proceed until an external dependency is resolved | Task checkboxes; include what it's blocked on |
| `#needs-review` | Requires review from another team member or stakeholder | Task checkboxes |
| `#follow-up` | Not urgent, but should be revisited in a future session | Task checkboxes |

**Rules:**
- Tags go on the checkbox line, not headings: `- [ ] Fix the thing #post-deploy`
- Only use defined tags — don't invent ad-hoc tags
- Tags are for tasks, not for categorization (that's what `type:` frontmatter is for)

## Backlinks

Backlinks are critical for discoverability. An AI scanning the vault uses backlinks to trace relationships between files. Always use `[[wikilinks]]` — the more connections, the easier it is to gather context.

Use `obsidian vault="X" backlinks path="<path>"` to discover what links to a file.

### Required structural backlinks
These are mandatory and create the core navigation graph:
- Daily log → session logs: `[[<project>/Session Logs/...|description]]`
- Daily log → project overviews: `[[<project>/Project Overviews/<Project Name> Overview|Project Name]]`
- Session log → plan (frontmatter): `plan: "[[<project>/Design Specs/...]]"`
- Plan superseded → replacement: `superseded_by: "[[...]]"`
- Plan replacement → predecessor: `Supersedes: [[...]]`
- Overview → technical reference (if exists): link in Ecosystem section
- Overview → related projects: `[[wikilinks]]` in Ecosystem section

### Contextual backlinks
Add these throughout body content whenever referencing something that exists in the vault:
- **Component docs**: `[[Resources/Data Grid]]`, `[[Resources/Date Picker]]`
- **Cross-project references**: `[[Onboarding/Project Overviews/Onboarding Overview|Onboarding]]`
- **Related sessions**: `[[API Project/Session Logs/Auth refactor + tests|yesterday's session]]`
- **Other projects' plans**: if a plan in one project informed decisions in another, link it

### In overviews
- Link to related projects in the Ecosystem section
- Link to the technical reference doc (if one exists) in the Ecosystem section
- Do NOT maintain a Plans table — plans are discoverable via backlinks

## Starting a Session

**Checklist — run at session start:**

- [ ] **Find and read the most recent daily log:**
  ```bash
  obsidian vault="X" files folder="Daily Logs"
  obsidian vault="X" read path="Daily Logs/<most recent>.md"
  ```

- [ ] **Read the project overview** for any project you'll be working on:
  ```bash
  obsidian vault="X" read path="<project>/Project Overviews/<Project Name> Overview.md"
  ```

- [ ] **Verify claims**: cross-check the daily log's "Open Threads" and project status against the overview and actual code/instance state. If anything is stale, correct the overview **before** reporting status to the user (use Read + Edit for mid-file updates).

- [ ] **Scan stale checkboxes** across all active project files:
  ```bash
  obsidian vault="X" tasks todo verbose
  ```
  Check off any items completed in prior sessions:
  ```bash
  obsidian vault="X" task done path="<path>" line=<n>
  ```
  Update plan statuses to `completed` if all steps are done:
  ```bash
  obsidian vault="X" property:set name="status" value="completed" path="<path>"
  ```

- [ ] **If a new day, create today's session log:**
  ```bash
  obsidian vault="X" create path="<project>/Session Logs/<description>.md" template="Session Log"
  obsidian vault="X" property:set name="sn_sys_id" value="" path="<project>/Session Logs/<description>.md"
  obsidian vault="X" property:set name="sn_synced" value="false" path="<project>/Session Logs/<description>.md"
  obsidian vault="X" property:set name="sn_category" value="session_log" path="<project>/Session Logs/<description>.md"
  obsidian vault="X" property:set name="project" value="<Project Name>" path="<project>/Session Logs/<description>.md"
  obsidian vault="X" property:set name="author" value="<author>" path="<project>/Session Logs/<description>.md"
  obsidian vault="X" property:set name="sn_project" value="<Project Name>" path="<project>/Session Logs/<description>.md"
  ```

**Why verify:** Daily logs and overviews drift. A previous session may have finished work but not updated the overview. Always read the source of truth (overview + code) and fix discrepancies before summarizing.

## During a Session

**Task tracking in plans:**
- Plans should have checkboxes (`- [ ]`) for each discrete step
- Check off steps **immediately** after completing — not at wrap-up:
  ```bash
  obsidian vault="X" task done path="<project>/Design Specs/<plan>.md" line=<n>
  ```
- When all plan steps done:
  ```bash
  obsidian vault="X" property:set name="status" value="completed" path="<project>/Design Specs/<plan>.md"
  ```
- `#post-deploy` tasks stay open until confirmed on target instance

**Task tracking in session logs:**
- Use an **Open Issues** section with checkboxes for work identified but not completed
- Check off items as they're resolved (even across sessions — go back and check off old items)

**Appending to session logs:**
```bash
obsidian vault="X" append path="<project>/Session Logs/<session>.md" content="## <heading>\n\n<content>"
```

**Keeping the overview current:**
- After completing a feature or phase: update the Status section immediately (use Read + Edit)
- Don't defer overview updates to wrap-up — do it while the context is fresh
- The status should be a single scannable sentence
- Keep overviews focused on what/how/ecosystem — never add field tables, API lists, or implementation details (those go in the technical reference doc)

## Wrapping Up

**Checklist — run at session end (every item is mandatory):**

### 1. Verify completed work
- [ ] Re-read every overview you touched this session
- [ ] Confirm status lines match what was actually accomplished (not what was planned)
- [ ] **Check off stale checkboxes across all active projects:**
  ```bash
  obsidian vault="X" tasks todo verbose
  ```
  Any item completed (this session or prior) gets checked off. Update plan `status: completed` when all steps are done.
- [ ] Scan for stale claims: "needs testing" when tests passed, "in progress" when completed, phases listed as upcoming that are done

### 2. Session log
- [ ] Append final summary of changes:
  ```bash
  obsidian vault="X" append path="<project>/Session Logs/<session>.md" content="## Summary\n\n<summary>"
  ```
- [ ] List all changed files
- [ ] Add **Open Issues** section with checkboxes for unfinished work
- [ ] Set status to complete:
  ```bash
  obsidian vault="X" property:set name="status" value="complete" path="<project>/Session Logs/<session>.md"
  ```

### 3. Daily log
- [ ] Create from template:
  ```bash
  obsidian vault="X" create path="Daily Logs/<brief description>.md" template="Daily Log"
  obsidian vault="X" property:set name="sn_sys_id" value="" path="Daily Logs/<name>.md"
  obsidian vault="X" property:set name="sn_synced" value="false" path="Daily Logs/<name>.md"
  obsidian vault="X" property:set name="sn_category" value="daily_log" path="Daily Logs/<name>.md"
  obsidian vault="X" property:set name="author" value="<author>" path="Daily Logs/<name>.md"
  obsidian vault="X" property:set name="projects" value="Project1,Project2" type=list path="Daily Logs/<name>.md"
  ```
- [ ] Append session links and project summaries:
  ```bash
  obsidian vault="X" append path="Daily Logs/<name>.md" content="## Sessions\n\n- [[<project>/Session Logs/<session>|description]]\n\n## Projects Active\n\n### [[<project>/Project Overviews/<Name> Overview|Project Name]]\n<verified current state>\n\n## Open Threads / Next Steps\n\n- <what the next session picks up>"
  ```
- [ ] Summarize each project's **verified** current state (freshly confirmed, not copied from a previous daily log)

### 4. Standup notes
- [ ] Create if it doesn't exist:
  ```bash
  # Check if today's standup exists
  obsidian vault="X" read path="Standups/YYYY-MM-DD.md"

  # If not found, create from template:
  obsidian vault="X" create path="Standups/YYYY-MM-DD.md" template="Standup"
  obsidian vault="X" property:set name="sn_sys_id" value="" path="Standups/YYYY-MM-DD.md"
  obsidian vault="X" property:set name="sn_synced" value="false" path="Standups/YYYY-MM-DD.md"
  obsidian vault="X" property:set name="sn_category" value="standup" path="Standups/YYYY-MM-DD.md"
  ```
- [ ] **One section per author.** Read the standup first. If the author already has a `### <author>` section, use Edit to update it in place — merge new bullets into "Today", update "Next" and "Blockers". Do NOT append a new heading like `### Alice (Evening)` or `### Alice (Late)`.
  ```bash
  # If author section does NOT exist yet, append it:
  obsidian vault="X" append path="Standups/YYYY-MM-DD.md" content="### <author>\n\n**Yesterday:**\n- Bullet points\n\n**Today:**\n- Next steps\n\n**Blockers:** None"

  # If author section ALREADY exists, use Read + Edit to update in place
  ```
- [ ] Do not overwrite other authors' sections

**Standard format (Monday-Friday, no weekend work):**
```
### <author>

**Yesterday:**
- Bullet points from session logs

**Today:**
- Next steps / open threads

**Blockers:** None (or list any)
```

**Weekend/multi-day format:** If work was done across multiple days since the last standup, list each day separately instead of "Yesterday":
```
### <author>

**Friday (2026-03-28):**
- Bullet points from Friday's session logs

**Saturday (2026-03-29):**
- Bullet points from Saturday's session logs

**Today:**
- Next steps / open threads

**Blockers:** None (or list any)
```

**Rules:**
- Check daily logs since the last standup date — if multiple days have logs, include all of them
- Only include days that have session logs (skip days with no work)
- On Monday with no weekend work, "Yesterday" means Friday — no special handling needed
- Each author owns their section — append yours, never edit another author's section

### 5. Archive handling
**No more moving files to Archive/.** Archived status is handled via frontmatter:
```bash
obsidian vault="X" property:set name="sn_tags" value="archived" path="<path>"
obsidian vault="X" property:set name="status" value="completed" path="<path>"
```
Sync plugin handles archive status if configured — no file moves needed.

### 6. Post-session verification
- [ ] Backlinks in active files point to correct paths
- [ ] No overview contains stale status claims or implementation details
- [ ] All new/modified files have `sn_synced: false` (plugin handles this automatically on edit)

## Project Lifecycle

| Event | Action |
|-------|--------|
| New project | Create from template, then set properties: |

```bash
# Create overview
obsidian vault="X" create path="<project>/Project Overviews/<Project Name> Overview.md" template="Project Overview"
obsidian vault="X" property:set name="sn_sys_id" value="" path="<project>/Project Overviews/<Project Name> Overview.md"
obsidian vault="X" property:set name="sn_synced" value="false" path="<project>/Project Overviews/<Project Name> Overview.md"
obsidian vault="X" property:set name="sn_category" value="project_overview" path="<project>/Project Overviews/<Project Name> Overview.md"
obsidian vault="X" property:set name="project" value="<Project Name>" path="<project>/Project Overviews/<Project Name> Overview.md"
obsidian vault="X" property:set name="sn_project" value="<Project Name>" path="<project>/Project Overviews/<Project Name> Overview.md"
```

| Event | Action |
|-------|--------|
| New area | Same as project but set `type: area` — areas are ongoing responsibilities, never "complete" |
| Project completes | `property:set name="status" value="complete"` + `property:set name="sn_tags" value="complete"` |
| New/modified component | Update `Resources/<name>.md` |

## Fallback Mode: CLI Unavailable

If Obsidian is not running or the CLI is not accessible, fall back to built-in tools:

| CLI operation | Fallback |
|---------------|----------|
| `create template=` | Write tool — include full frontmatter + body sections manually |
| `property:set` | Read + Edit — modify the YAML frontmatter block directly |
| `property:read` | Read tool — parse the YAML frontmatter |
| `append` | Read + Edit — add content at the end of the file |
| `tasks todo` | Grep for `- \[ \]` across vault files |
| `task done` | Read + Edit — change `- [ ]` to `- [x]` on the target line |
| `files folder=` | Glob for `<folder>/**/*.md` |
| `search query=` | Grep for the query text |
| `backlinks` | Grep for `[[filename` across vault files |
| `move` | Bash `mv` (note: won't auto-update wikilinks) |

All frontmatter schemas, naming conventions, and structural rules still apply regardless of mode.
