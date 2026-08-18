---
name: checkpoint-resume
description: Resume work from a specific checkpoint id, deterministically. Trigger on /checkpoint-resume, "/checkpoint-resume <id>", "resume checkpoint <id>", "pick up k7f3", or a bare resume request when you want to choose from live checkpoints. Reads the vault-wide _CHECKPOINTS.md registry to land on the EXACT thread — never a mtime guess.
---

# checkpoint-resume

Dedicated resume entry point for the checkpoint discipline. Takes an optional **4-char
checkpoint id** and resumes that exact thread.

This skill is the thin front door; the full reconcile + staleness logic lives in the
**`checkpoint`** skill's *Resume* section — follow it there. This file exists so `/checkpoint-resume
<id>` is a first-class command.

## With an id — `/checkpoint-resume k7f3`

1. Open `<vault-root>/_CHECKPOINTS.md`, find the row whose `id` matches.
   - **Not found** → say so, then list the `status: live` rows (newest first) and ask which.
2. Read the `pointer` file that row names (`<project>/Session Logs/_RESUME-<id>.md`).
3. Hand off to the **`checkpoint`** skill's *Resume* steps 2–5: reconcile the fingerprint against
   live state **for the surfaces the pointer's `surfaces:` field names** (git, ServiceNow, both, or
   neither — do not run the other paradigm's commands), apply the staleness guard, re-hydrate
   working state, **resolve the pointer's status** (`complete` on resume, `stale` if abandoned),
   then restate where things stand and the NEXT action and continue.

   Two things that decide the call and are easy to skim past:
   - **ServiceNow has its own staleness signals** — a record edited after `checkpoint_at`, an
     update set now `Complete`/`Ignored`, a different `instance`. The guard's git signals being
     inapplicable is **not** evidence a pointer is fresh.
   - **Step 2 gates step 3.** A drifted current-set pointer alone is routine re-hydration; a
     closed set, a changed record, or a different instance is staleness. Don't repair your way
     past a fired signal.

## Without an id — `/checkpoint-resume`

1. Read `<vault-root>/_CHECKPOINTS.md`. List the `status: live` rows newest-first — show `id`,
   `project`, `checkpoint_at`, and the one-line `NEXT`.
2. Ask the user which id to resume. **Never auto-pick**, and never fall back to sorting
   `_RESUME.md` files by mtime — that is the exact failure the id system removes.
3. On their choice, proceed as *With an id* above.

## Notes

- If `_CHECKPOINTS.md` doesn't exist, no checkpoint was ever written with this version — tell the
  user plainly and reconstruct current state from the live surfaces (git, ServiceNow, or whatever
  the work actually lives on), don't narrate an old `_RESUME.md` as live.
- `status` is `live` | `complete` | `stale`. Bare `/checkpoint-resume` lists only `live` rows.
- Ids are minted by the `checkpoint` skill (4 chars, alphabet `23456789abcdefghjkmnpqrstuvwxyz`).
