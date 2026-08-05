# Fallback Mode: CLI Unavailable

Reach for this only when Obsidian isn't running or the CLI binary can't be found. Map each CLI operation to built-in tools:

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
| `move` | Bash `mv` (won't auto-update wikilinks) |

All frontmatter schemas (SCHEMAS.md), naming conventions, and structural rules (REFERENCE.md) still apply regardless of mode.
