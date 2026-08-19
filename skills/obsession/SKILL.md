---
name: obsession
description: Use when starting a work session (to load recent project context), wrapping one up, or when the user asks to write session or standup notes, lint the vault, or rebuild the index.
---

# Obsession

Track work sessions, daily progress, and standup notes in an Obsidian vault.

Reference lives in three disclosed files, loaded on demand:
- **[SCHEMAS.md](SCHEMAS.md)** — frontmatter schema for every file type. Read before creating or editing a file's frontmatter.
- **[REFERENCE.md](REFERENCE.md)** — vault structure, naming conventions, tags, backlink rules. Read when placing a new file or wiring links.
- **[FALLBACK.md](FALLBACK.md)** — built-in-tool equivalents for every CLI command. Read only when the CLI is unavailable.

## CLI Configuration

This skill uses the **Obsidian CLI** (bundled with Obsidian 1.12+) as its primary interface.

- **CLI binary** — detect by platform:
  - **macOS:** `/Applications/Obsidian.app/Contents/MacOS/obsidian`
  - **Windows:** `%LOCALAPPDATA%\Obsidian\Obsidian.exe`
  - **Linux:** `obsidian` on PATH, else `/usr/bin/obsidian`, `/snap/bin/obsidian`
  - If none found, try `obsidian` as a bare command
- **Vault name / path** — read from CLAUDE.md → `Obsidian Vault Integration`

All commands use `vault="<name>"` (e.g., `vault="My Vault"`). If the CLI is unavailable (Obsidian not running, binary not found), switch to [FALLBACK.md](FALLBACK.md).

### Author Detection

The current user's identity comes from the Snobby sync plugin. At session start, read:

```bash
cat "<vault_path>/.obsidian/plugins/snobby/data.json"
```

Extract `settings.userDisplayName` and use the **first name only** (split on space, index 0) as `<author>` everywhere — session logs, daily logs, standup sections. Example: `"Caleb Bonitz"` → `Caleb`. If snobby settings are missing or empty, fall back to `$CLAUDE_AUTHOR`; if neither exists, prompt the user for their name before creating authored content.

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
- **Mid-file edits** need Read + Edit tools (the CLI has no edit/replace command)

## The create recipe

A file made with `create template=` inherits **all** of the template's frontmatter, including its sync identifiers (`sn_sys_id`, `sn_category: template`). Left as-is, the new file collides with the template's remote record and lands in the wrong folder on sync. So **immediately after every `create template=`**, reset the three sync fields:

```bash
obsidian vault="X" property:set name="sn_sys_id" value="" path="<path>"
obsidian vault="X" property:set name="sn_synced" value="false" path="<path>"
obsidian vault="X" property:set name="sn_category" value="<category>" path="<path>"
```

Then set the file's own dynamic properties (`project`, `author`, `sn_project`, …) per its schema in [SCHEMAS.md](SCHEMAS.md). Wherever a step below says **run the create recipe with `sn_category=<x>`**, it means these three resets plus the type's dynamic properties. If the vault has no sync plugin, the resets are harmless no-ops — still run them.

**Templates** live in `Templates/`: Session Log, Daily Log, Design Spec, Standup, Project Overview, Component Doc, Index. A template's `{{date}}` placeholder resolves automatically on `create`.

## Starting a Session

**Checklist — run at session start:**

- [ ] **Read the vault index** for a fast overview:
  ```bash
  obsidian vault="X" read path="Index/index.md"
  ```
  If it doesn't exist yet, skip — you'll create it at wrap-up.

- [ ] **Read the most recent daily log:**
  ```bash
  obsidian vault="X" files folder="Daily Logs"
  obsidian vault="X" read path="Daily Logs/<most recent>.md"
  ```

- [ ] **Read the latest session log's Resume pointer** for the project(s) you're continuing — it's the handoff from the last session (current update set/branch, what landed, the next action, frontier). Read it before exploring code, and verify its claims against real state before building on them:
  ```bash
  obsidian vault="X" files folder="<project>/Session Logs"
  obsidian vault="X" read path="<project>/Session Logs/<most recent>.md"
  ```

- [ ] **Read the project overview** for each project you'll touch:
  ```bash
  obsidian vault="X" read path="<project>/Project Overviews/<Project Name> Overview.md"
  ```

- [ ] **Verify claims.** Cross-check the daily log's "Open Threads" and project status against the overview and actual code/instance state. Overviews and logs drift — a prior session may have finished work without updating the overview. Correct the overview (Read + Edit for mid-file changes) **before** reporting status to the user.

- [ ] **Scan stale checkboxes** across all active project files, then check off anything already done and close finished plans:
  ```bash
  obsidian vault="X" tasks todo verbose
  obsidian vault="X" task done path="<path>" line=<n>
  obsidian vault="X" property:set name="status" value="completed" path="<path>"
  ```

- [ ] **If a new day, create today's session log** (see [SCHEMAS.md](SCHEMAS.md) → Session Log):
  ```bash
  obsidian vault="X" create path="<project>/Session Logs/<description>.md" template="Session Log"
  # run the create recipe with sn_category=session_log, then:
  obsidian vault="X" property:set name="project" value="<Project Name>" path="<...>.md"
  obsidian vault="X" property:set name="author" value="<author>" path="<...>.md"
  obsidian vault="X" property:set name="sn_project" value="<Project Name>" path="<...>.md"
  ```

## During a Session

**Task tracking in plans:**
- Plans carry a checkbox (`- [ ]`) per discrete step
- Check off each step **immediately** after completing it — not at wrap-up:
  ```bash
  obsidian vault="X" task done path="<project>/Design Specs/<plan>.md" line=<n>
  ```
- When every step is done: `property:set name="status" value="completed"`
- `#post-deploy` tasks stay open until confirmed on the target instance

**Task tracking in session logs:**
- Keep an **Open Issues** section with checkboxes for work identified but not finished
- Check items off as they resolve, even across sessions — go back and close old ones

**Writing to session logs.** Templates ship placeholder headers (`## Summary`, `## Changes`, …). The CLI has no find/replace, so fill a placeholder **in place with the Read + Edit tools** — replace `## Summary\n` with the header plus its content. Reserve the CLI's `append` for genuinely new sections not in the template (appending an existing header would duplicate it):
```bash
# new sections only:
obsidian vault="X" append path="<session>.md" content="## <new heading>\n\n<content>"
```

**Filing valuable synthesis.** When a conversation produces reusable analysis — cross-project comparisons, architectural decisions, research findings — save it to the vault rather than let it die in chat. Use a Design Spec (`type: plan`) for project-specific analysis, a Resource (`type: component`) for cross-project knowledge. Ask first: "This analysis seems worth keeping — want me to save it to the vault?"

**Keeping the overview current.** After finishing a feature or phase, update the Status section immediately (Read + Edit) while context is fresh — don't defer to wrap-up. Status is a single scannable sentence. Keep overviews to what/how/ecosystem; field tables, API lists, and implementation detail go in the technical reference doc (see [SCHEMAS.md](SCHEMAS.md) → Project Overview).

## Wrapping Up

**Checklist — run at session end (every item is mandatory):**

### 1. Verify completed work
- [ ] Re-read every overview you touched this session
- [ ] Confirm status lines match what was actually accomplished, not what was planned
- [ ] Check off stale checkboxes across all active projects (`tasks todo verbose`); set plan `status: completed` when all steps are done
- [ ] Catch stale claims: "needs testing" when tests passed, "in progress" when done, phases listed as upcoming that are finished

### 2. Session log
- [ ] Fill each placeholder section — Summary, Changes, Open Issues, Changed Files, **Resume pointer** — **in place with the Read + Edit tools** (the CLI has no find/replace; `append` would duplicate the existing headers). Replace `## Summary\n` with the header plus its content, and so on for each.
- [ ] **The Resume pointer is the handoff.** Fill it machine-crisp so a fresh session (after `/clear`, zero memory of this conversation) can continue without re-deriving: current update set/branch/working state, what landed and where, the single NEXT action, the frontier, and any pending verification (human test, review, deploy). Verify each line against real state — don't write what you hoped happened. (See the `checkpoint` skill for the same discipline as a standalone trigger.)
- [ ] Set status complete: `obsidian vault="X" property:set name="status" value="complete" path="<session>"`

### 3. Daily log
- [ ] Create from template (see [SCHEMAS.md](SCHEMAS.md) → Daily Log):
  ```bash
  obsidian vault="X" create path="Daily Logs/<brief description>.md" template="Daily Log"
  # run the create recipe with sn_category=daily_log, then:
  obsidian vault="X" property:set name="author" value="<author>" path="Daily Logs/<name>.md"
  obsidian vault="X" property:set name="projects" value="Project1,Project2" type=list path="Daily Logs/<name>.md"
  ```
- [ ] Append session links and **freshly verified** per-project state (not copied from a prior daily log):
  ```bash
  obsidian vault="X" append path="Daily Logs/<name>.md" content="## Sessions\n\n- [[<project>/Session Logs/<session>|description]]\n\n## Projects Active\n\n### [[<project>/Project Overviews/<Name> Overview|Project Name]]\n<verified current state>\n\n## Open Threads / Next Steps\n\n- <what the next session picks up>"
  ```

### 4. Standup notes

**Compute the standup date FIRST, before any file operation.** The standup file is dated for the meeting day = the **next business day after the work being recapped**.

- `<work_day>` = `currentDate` from the environment (the day work happened — typically today)
- `<standup_date>` = next business day after `<work_day>`, skipping Saturday, Sunday, and known holidays:
  - Work Mon → standup Tue; work Fri → standup **Mon**; work pre-holiday → standup first business day after
- If unsure whether a date is a holiday, ask the user before creating the file
- Use the computed `<standup_date>` as `YYYY-MM-DD` in every command below

- [ ] **Rule out an existing standup before creating** — one file per day is shared by the whole team, so a duplicate is easy to make when your local vault is behind the server. Resolve in order:

  1. **Local check** — is the file already in the vault?
     ```bash
     # Also catches collision variants like "2026-04-22 (abc123).md"
     ls "<vault_path>/Standups/" | grep "<standup_date>"
     ```
     Found → skip creation, go to the per-author step.

  2. **Not local → query the sync backend directly (authoritative).** If the vault syncs to a backend an available MCP can query (e.g. a ServiceNow MCP fronting the sync plugin's table), query it for a standup dated `<standup_date>` — this settles existence without a manual sync round-trip. Connection details (instance, table, field names) live in your environment config (CLAUDE.md), not here.
     - **Exists on server** → a teammate made it and you haven't pulled it. Tell the user who and when (from updated-by / updated-on), ask them to **sync the plugin to pull it down**, then continue to the per-author step so the plugin's merge reconciles your section. **STOP until the user confirms the sync.**
     - **Not on server** → safe to create (below).

  3. **No queryable backend → manual gate.** Ask: *"No standup for `<standup_date>` found locally and I can't check the server. Please sync and confirm no one else created it, then I'll create it."* **STOP until confirmed.**

  Create only once existence is ruled out:
  ```bash
  obsidian vault="X" create path="Standups/<standup_date>.md" template="Standup"
  # run the create recipe with sn_category=standup, then:
  obsidian vault="X" property:set name="date" value="<standup_date>" path="Standups/<standup_date>.md"
  ```

- [ ] **Own exactly one `### <author>` section; preserve everyone else's.** Read the standup first. If your section exists, Read + Edit to **add** bullets — never replace content, never append a second heading like `### Alice (Evening)`:
  - **Done / prior days:** keep existing bullets; add a new day block only for work on days not yet listed
  - **Next:** keep all existing bullets, append new ones for work since your last update
  - **Blockers:** update only when status changed
  ```bash
  # Section does NOT exist yet — append it (see Section shape below for the full recipe):
  obsidian vault="X" append path="Standups/<standup_date>.md" content="### <author>\n\n**Done (<work_day>):**\n\n- **Headline claim.**\n  - Evidence.\n\n**Next:**\n- Next steps\n\n**Blockers:** None"
  ```

**Section shape.** Every `### <author>` section is these four parts, in this order:

1. **Continuity line** — italic, one sentence, only when your previous entry was in an earlier standup. What that entry covered plus a wikilink to it, so a reader who missed a day can catch up without hunting.
2. **`**Done (<work_day>):**`** — the day's outcomes. Name the date; don't write "Yesterday".
3. **`**Next:**`** — what `<standup_date>` picks up.
4. **`**Blockers:**`** — `None`, or what you need and from whom.

**Every Done item is a headline plus its evidence.**

- The top-level bullet is **one bold sentence** carrying the outcome *and* the part that wasn't expected. A reader who stops there has the news.
- Indented under it: the numbers, the table/field/file names, the trap the next person would hit, the assumption that turned out wrong. All detail lives here — never in the headline.
- Blank line between top-level bullets. Sub-bullets stay tight under their headline.
- Deep dives link out instead of growing: `See [[<project>/Session Logs/<name>|the session log]]`.
- Group by outcome, not chronology. One bullet per outcome even if it spanned three sessions; one bullet per session is the shape to avoid.
- Corrections, reverts, and tooling gotchas get their own headline bullets — they are the highest-value lines in the note, not footnotes.
- Numbers over adjectives: `6,848 vs 2,893 records`, `765/765 across 217 groups`, `0 → 11 edges` — not "several" or "much better".

**Standard format** (one work day rolls into the standup — e.g. Tue recaps Mon):
```markdown
### <author>

*Previous entry — **2026-08-17**: contacts scope cuts, workbook import end to end, three validation fixes. Full write-up: [[Standups/2026-08-17|2026-08-17 standup]].*

**Done (2026-08-18):**

- **Dropped 8 legacy columns off the region table, and none of them needed the data migration the backlog assumed.**
  - All 16 regions carrying a dealer group already matched their master exactly; the 11 with IT data held test junk (`test/test/test`, `text me pls`).
  - The backlog was also wrong that the phase was unblocked — `Create Cyber Group` was still active and still writing all 8 values right up to the delete.

- **Two intake paths were silently discarding the rep-entered contact.** Name, email and phone were written to columns that no longer exist, and a write to a missing column is a no-op with no error.
  - Trap that would have cost a day: intake-written rows must be `u_manual = true`, or the nightly sweep deactivates them within 24 hours with only a `gs.warn` to show.

- **Workbook delivered for <customer>.** See [[VMS/Session Logs/service provider workbook|the session log]].
  - The finding underneath it: questionnaires attach two ways and the portal reads only one — 6,848 records via one column against 2,893 via the other.

- **Corrections on the record.** The nav offset shipped twice from CSS arithmetic without ever loading the page; measured properly, the original was already correct. All of it backed out.

**Next:**
- <what <standup_date> picks up — one line each, no sub-bullets needed>

**Blockers:** None (or: what you need, from whom)
```

**Multi-day format** — whenever more than one work day rolls in (Mon recapping Fri+weekend, post-holiday, …). One `**Done (<date>):**` block per day, same headline+evidence shape inside each:
```markdown
### <author>

**Done (2026-03-28, Friday):**

- **Headline claim.**
  - Evidence.

**Done (2026-03-31, Monday):**

- **Headline claim.**
  - Evidence.

**Next:**
- Plans for <standup_date>

**Blockers:** None (or list any)
```

- Check daily logs since the last standup; include every day that has session logs (skip days with none)
- One `Done` block fits only when exactly one work day rolls in — otherwise one block per day
- `Next` describes plans for `<standup_date>`, not actuals from `<work_day>`
- Source the headlines from the session logs' Summary and Resume pointer; if a session log has no headline-worthy outcome, it doesn't earn a bullet

### 5. Update the vault index
- [ ] Follow the **Incremental Update** workflow under Vault Index below

### 6. Archive handling
Archived status is frontmatter, not a folder move:
```bash
obsidian vault="X" property:set name="sn_tags" value="archived" path="<path>"
obsidian vault="X" property:set name="status" value="completed" path="<path>"
```
The sync plugin handles archive status if configured.

### 7. Post-session verification
- [ ] Backlinks in active files point to correct paths
- [ ] No overview holds stale status claims or implementation detail
- [ ] All new/modified files have `sn_synced: false` (the plugin sets this on edit)

## Vault Index

`Index/index.md` is a flat catalog of every significant file with a one-line summary — a single-read overview of the vault that saves the LLM from scanning every folder.

### Format

Entries grouped by type, one line each: wikilink + summary + status. Active items sort before completed within a group; alphabetical within groups.

```markdown
## Active Projects
- [[Project/Project Overviews/Project Overview|Project Name]] — One-line description (status)

## Active Areas
- [[Area/Project Overviews/Area Overview|Area Name]] — One-line description

## Active Design Specs
- [[Project/Design Specs/plan-name|plan-name]] — One-line purpose (status)

## Resources
- [[Resources/Component Name|Component Name]] — One-line description

## Recent Session Logs
- [[Project/Session Logs/description|description]] — YYYY-MM-DD, author

## Recent Daily Logs
- [[Daily Logs/description|description]] — YYYY-MM-DD

## Completed Projects
- [[Project/Project Overviews/Project Overview|Project Name]] — Completed YYYY-MM-DD
```

- One line per entry, summaries concise, written from the file's current content (not copied from stale sources)
- Session and daily logs: only the last 2 weeks — drop older entries on update
- Completed/archived projects: one line each, no sub-entries for their specs/sessions

### Incremental Update (at wrap-up)

Touch only entries for files created or modified this session.

1. Read `Index/index.md`
2. For each file created this session: add an entry in the right group
3. For each file modified this session: update its summary/status if changed
4. Remove any session/daily log entry older than 2 weeks
5. Edit individual lines with find/replace — do **not** rewrite the whole file

### Full Rebuild

Run only when lint reports significant index drift, or on explicit request ("rebuild the index"). Regenerates the file from scratch.

1. List all project folders (`files folder=` per top-level project)
2. Read frontmatter from every overview, plan, and resource doc
3. List recent session and daily logs (last 2 weeks)
4. Generate the full index in the format above and write the complete file

**First-time creation:**
```bash
obsidian vault="X" create path="Index/index.md" template="Index"
# If no Index template exists, use Write with full frontmatter, then:
# run the create recipe with sn_category=index
```
Then run the rebuild workflow to populate it.

## Project Lifecycle

| Event | Action |
|-------|--------|
| New project | Create overview from template, run the create recipe, then set `project` + `sn_project` (below) |
| New area | Same as a new project but set `type: area` — areas are ongoing, never "complete" |
| Project completes | `property:set status=complete` + `property:set sn_tags=complete` |
| New/modified component | Update `Resources/<name>.md` |

```bash
obsidian vault="X" create path="<project>/Project Overviews/<Project Name> Overview.md" template="Project Overview"
# run the create recipe with sn_category=project_overview, then:
obsidian vault="X" property:set name="project" value="<Project Name>" path="<...> Overview.md"
obsidian vault="X" property:set name="sn_project" value="<Project Name>" path="<...> Overview.md"
```

## Lint

A read-only audit of vault health. Scans, reports, asks which to fix — changes nothing until approved.

**Trigger:** "lint the vault", "vault lint", "check vault health", or similar.

### Checks

| Check | What it flags |
|-------|--------------|
| **Missing index entries** | Files in the vault not listed in `Index/index.md` |
| **Stale index entries** | Index entries pointing to files that no longer exist |
| **Orphan pages** | Files with zero backlinks |
| **Stale overviews** | Overview status claims that don't match actual project state |
| **Unclosed plans** | All tasks checked off but `status` still `active` |
| **Unclosed sessions** | `status: in-progress` on session logs from past days |
| **Missing frontmatter** | Required fields absent for the file's `type` |
| **Broken wikilinks** | Links to files that don't exist |
| **Missing required backlinks** | Daily logs without session links, sessions without plan links |

### Workflow

1. Run all checks, collecting issues into a grouped report
2. Print the report to the conversation:
   ```
   ## Vault Lint Report

   ### Missing Index Entries (3)
   - Project/Design Specs/new-plan.md
   - Resources/New Component.md
   - Project/Session Logs/recent session.md

   ### Unclosed Sessions (1)
   - Project/Session Logs/old session.md — status: in-progress, date: 2026-04-25

   ✅ No issues: orphan pages, broken wikilinks, missing frontmatter
   ```
3. Ask: "Want me to fix all, some, or none of these?"
4. Fix only what the user approves:
   - **Missing index entries** → add to index
   - **Stale index entries** → remove from index
   - **Unclosed sessions** → set `status: complete`
   - **Unclosed plans** → set `status: completed`
   - **Missing frontmatter** → add required fields with sensible defaults
   - **Stale overviews** → read current state, update the status line
   - **Broken wikilinks / orphan pages / missing backlinks** → report only; the user decides intent

### What lint does NOT do
- Rewrite the index (that's Full Rebuild)
- Modify file content beyond frontmatter fixes
- Delete files
- Make any change without asking
