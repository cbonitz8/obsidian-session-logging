# Obsidian Session Logging

A Claude Code skill for tracking work sessions, daily progress, and standup notes in an Obsidian vault. Uses the Obsidian CLI (1.12+) for vault operations with fallback to file tools when CLI is unavailable.

## What It Does

- **Session logs** — per-project work session tracking with changed files, open issues, and plan references
- **Daily logs** — cross-project daily summaries with links to session logs and open threads
- **Standups** — shared daily standup notes with per-author sections
- **Project overviews** — living status documents kept current as work progresses
- **Design specs** — plan tracking with status lifecycle (active → completed → superseded)
- **Wrap-up checklists** — end-of-session verification to prevent stale documentation

## Installation

### Claude Code

```bash
claude plugins add obsidian-session-logging@cbonitz8
```

Or add to your `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "obsidian-session-logging@cbonitz8": true
  },
  "extraKnownMarketplaces": {
    "cbonitz8": {
      "source": {
        "source": "github",
        "repo": "cbonitz8/obsidian-session-logging"
      }
    }
  }
}
```

## Configuration

The skill reads vault configuration from your project's `CLAUDE.md` file. Add this section:

```markdown
## Obsidian Vault Integration

- **Vault name:** My Vault
- **Vault path:** /path/to/your/vault
```

### Optional: Sync plugin fields

If you use a sync plugin (e.g., Snobby for ServiceNow sync), the skill manages `sn_`-prefixed frontmatter fields for sync state tracking. These fields are optional — the skill works without them, but if present, it handles them correctly:

- `sn_sys_id` — remote record identifier
- `sn_category` — maps to remote document category
- `sn_project` — maps to remote project field
- `sn_tags` — comma-separated tags
- `sn_synced` — sync status flag

If you don't use a sync plugin, you can ignore these fields entirely.

## Vault Structure

The skill expects this folder layout:

```
<vault>/
  <project>/
    Project Overviews/
      <Project Name> Overview.md
    Session Logs/
      <description>.md
    Design Specs/
      <plan-name>.md
  Daily Logs/
    <description>.md
  Standups/
    YYYY-MM-DD.md
  Resources/
    <Component Name>.md
  Templates/
```

- Projects are **top-level folders**
- Daily Logs and Standups are **top-level** (not per-project)
- Session Logs and Design Specs live **under their project folder**

## Templates

The skill uses Obsidian templates for creating new files. Create these templates in your vault's `Templates/` folder:

- **Session Log** — frontmatter: project, date, type, author, status
- **Daily Log** — frontmatter: date, type, author, projects
- **Design Spec** — frontmatter: project, type, status, date, author
- **Standup** — frontmatter: date, type
- **Project Overview** — frontmatter: project, status, type
- **Component Doc** — frontmatter: type, name

See the [SKILL.md](skills/obsidian-session-logging/SKILL.md) for complete frontmatter schemas.

## Usage

The skill activates automatically when you:

- Start a new conversation (reads recent context)
- Ask to update logs, create session entries, or generate standup notes
- Wrap up a work session

### Slash command

```
/obsidian-session-logging
```

### Example workflows

**Starting work:**
> "Let's work on Project X"
>
> Skill reads the latest daily log and project overview, verifies status claims, and reports current state.

**Wrapping up:**
> "Let's wrap up"
>
> Skill runs the wrap-up checklist: updates overview status, finalizes session log, creates daily log with verified state, updates standup notes.

## Requirements

- **Obsidian 1.12+** with CLI support
- Claude Code CLI or compatible agent

## CLI Compatibility

The skill auto-detects the Obsidian CLI binary:

| Platform | Default path |
|----------|-------------|
| macOS | `/Applications/Obsidian.app/Contents/MacOS/obsidian` |
| Windows | `%LOCALAPPDATA%\Obsidian\Obsidian.exe` |
| Linux | `obsidian` on PATH, `/usr/bin/obsidian`, `/snap/bin/obsidian` |

If Obsidian is not running or the CLI is unavailable, the skill falls back to direct file operations (Read/Write/Edit/Glob/Grep).
