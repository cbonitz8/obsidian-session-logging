# Vault Reference

Structure, naming, tags, and linking rules. Consult when creating or placing files, or resolving where something lives.

## Vault Structure

Read the vault path from the project's CLAUDE.md or memory. Expected layout:

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
  Index/                          # Vault-wide file catalog for LLM navigation
    index.md
  Resources/                      # Reusable component docs
    Component Name.md
  Templates/                      # Obsidian templates
  Project Overviews/
    ✍️ Dashboard.md               # Rolling dashboard with dataview queries
```

**Structure rules:**
- Projects are **top-level folders** — no `Projects/` prefix
- Daily Logs are **top-level** — no `Areas/` prefix
- Overviews live at `<project>/Project Overviews/<Project Name> Overview.md`
- Component docs live directly in `Resources/` — no `Components/` subfolder
- Folders are `Session Logs/` (not `Sessions/`) and `Design Specs/` (not `Plans/`)
- **No Archive folder** — archived docs stay in their project folder, marked via frontmatter (`sn_tags: "archived"` or sync-plugin tags)

## Naming Conventions

| File Type | Pattern | Example |
|-----------|---------|---------|
| Session log | `<brief description>.md` | `API refactor + auth flow.md` |
| Daily log | `<brief description>.md` | `API refactor + deployment.md` |
| Design spec | `<kebab-name>.md` | `auth-migration-plan.md` |
| Standup | `YYYY-MM-DD.md` | `2026-04-06.md` |
| Project overview | `<Project Name> Overview.md` | `My Project Overview.md` |
| Component doc | `<Display Name>.md` | `Data Grid Component.md` |
| Index | `index.md` | `index.md` |

- Dates live in the `date:` frontmatter property, not the filename. Files with dates in the name are legacy; new files omit the date.
- Session/daily descriptions: lowercase, concise, `+` to join topics
- Plan names: kebab-case
- Overview filenames carry the project prefix so they stay distinguishable in flat views (search, browser)

## Tags

Tags mark **action items** on task checkboxes. Keep the set small — every tag powers a query or workflow.

| Tag | Meaning | Use on |
|-----|---------|--------|
| `#post-deploy` | Requires action on the target instance after code deploy | Task checkboxes in session logs, plans |
| `#blocked` | Cannot proceed until an external dependency resolves | Task checkboxes; include what it's blocked on |
| `#needs-review` | Requires review from another team member or stakeholder | Task checkboxes |
| `#follow-up` | Not urgent, revisit in a future session | Task checkboxes |

- Tags go on the checkbox line, not headings: `- [ ] Fix the thing #post-deploy`
- Use only the defined tags
- Tags mark tasks, not categories — categorization is `type:` frontmatter

## Backlinks

An AI scanning the vault traces relationships through backlinks. Use `[[wikilinks]]` wherever you name something that exists in the vault — more connections, faster context.

Discover what links to a file: `obsidian vault="X" backlinks path="<path>"`

**Required structural backlinks** (the core navigation graph):
- Daily log → session logs: `[[<project>/Session Logs/...|description]]`
- Daily log → project overviews: `[[<project>/Project Overviews/<Project Name> Overview|Project Name]]`
- Session log → plan (frontmatter): `plan: "[[<project>/Design Specs/...]]"`
- Plan superseded → replacement: `superseded_by: "[[...]]"`
- Plan replacement → predecessor: `Supersedes: [[...]]`
- Overview → technical reference (if it exists) and related projects: `[[wikilinks]]` in the Ecosystem section

**Contextual backlinks** — add throughout body content when referencing vault items:
- Component docs: `[[Resources/Data Grid]]`, `[[Resources/Date Picker]]`
- Cross-project references: `[[Onboarding/Project Overviews/Onboarding Overview|Onboarding]]`
- Related sessions: `[[API Project/Session Logs/Auth refactor + tests|yesterday's session]]`
- Other projects' plans that informed a decision

Overviews link to related projects and the technical reference in the Ecosystem section. Overviews do not maintain a Plans table — plans surface via backlinks.
