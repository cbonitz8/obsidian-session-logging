---
name: checkpoint
description: Use to checkpoint work before clearing context and to resume after. Trigger on /checkpoint, "checkpoint", "save progress before I clear", "wrap up so I can /clear", "resume from where we left off", "pick up the last checkpoint". Bridges the update-log -> /clear -> resume loop by writing a machine-crisp Resume pointer addressed by a short checkpoint id, registered in a vault-wide index so resume lands on the EXACT thread, never a guess.
---

# checkpoint

Bridges the **update session log → `/clear` → resume** loop. Two directions; pick by
what the user is doing.

The `/clear` itself is a harness command only the human runs — this skill CANNOT clear
its own context. So the checkpoint direction ends by telling the user it's safe to clear;
the resume direction starts a fresh session from the written pointer.

## Why ids: two failures a single per-project pointer caused

Earlier this skill wrote ONE `<project>/Session Logs/_RESUME.md` per project and resume
guessed which project by recency. That produced two silent failures:

1. **Wrong-thread resume across projects** — recency picked a different project's pointer.
2. **Intra-project clobber** — a project with two live threads had the second checkpoint
   *overwrite* the first thread's pointer, destroying it.

The fix: every checkpoint is **addressable by a short id** and registered in a **vault-wide
index**, and each id gets its **own durable pointer file** that no other thread overwrites.
Resume by id lands on the exact thread; resume without an id lists the live ones and asks —
it never guesses by mtime.

## The three artifacts

1. **Per-id pointer** — `<project>/Session Logs/_RESUME-<id>.md`. The durable, high-fidelity
   handoff for ONE thread. Never clobbered by another thread (different id = different file).
   Schema below.
2. **Vault index** — `<vault-root>/_CHECKPOINTS.md`. One row per checkpoint id. The
   authoritative "what can I resume, and where does it live" registry. Resume reads THIS, not
   mtime. Do not hand-edit ids.
3. **Latest alias** — `<project>/Session Logs/_RESUME.md`. Overwritten each checkpoint with a
   copy of the newest per-id pointer for that project. Back-compat convenience only ("resume
   this project's most recent"); the index + per-id files are authoritative.

### Checkpoint id

A **4-character** id from the unambiguous alphabet `23456789abcdefghjkmnpqrstuvwxyz`
(no `0 1 o l i`), e.g. `k7f3`, `q9m2`, `h4tx`. Mint one per NEW thread. Before using a freshly
minted id, grep `_CHECKPOINTS.md` for it and regenerate on the rare collision. **If this
session was itself resumed from an id, reuse that same id** when re-checkpointing the same
thread — do not mint a new one (keeps one thread = one id across its whole life).

### Per-id pointer schema (`_RESUME-<id>.md`)

```markdown
---
type: resume-pointer
id: <4-char id>
project: <Project>
checkpoint_at: <YYYY-MM-DD HH:MM>        # when this pointer was written
session_log: "[[<full session-log note title>]]"
status: live | complete                   # live = work in flight; complete = nothing to resume
# --- working-state fingerprint (fill whichever apply) ---
git_branch: <branch or n/a>
git_head: <short sha or n/a>
git_ahead_of_main: <N or n/a>
git_dirty: <true/false or n/a>
update_set: "<name> (<sys_id>)" or n/a
---

NEXT: <the single first action the resuming session should take>

<the full Resume pointer block: Working state / Landed / NEXT / Frontier / Pending verification>
```

### Index schema (`<vault-root>/_CHECKPOINTS.md`)

```markdown
# Checkpoints — resume registry (authoritative; do not hand-edit ids)

| id | checkpoint_at | status | project | session_log | pointer |
|----|---------------|--------|---------|-------------|---------|
| k7f3 | 2026-08-04 15:30 | live | EIS 2.0 | [[sync orchestrator engine + employee specs]] | EIS 2.0/Session Logs/_RESUME-k7f3.md |
```

Newest row on top. One row per id — **upsert** (update the existing row when re-checkpointing
that id; append a new row for a new id). Never delete rows on checkpoint; resume flips status.

## Checkpoint (before `/clear`)

Goal: leave a pointer precise enough that a fresh session (zero memory of this conversation)
can continue without re-deriving anything.

1. **Capture live state FIRST, before writing anything.** Run the real commands so the
   fingerprint is true, not remembered: `git branch --show-current`, `git rev-parse --short HEAD`,
   `git status --short`, ahead-count (`git rev-list --count main..HEAD` or vs. the PR base),
   and — for SN work — `sndeck us get`. You cannot write an honest pointer from memory.
2. **Determine the id.** Reuse the id this session was resumed from, if any; otherwise mint a
   new 4-char id and confirm it's absent from `_CHECKPOINTS.md`.
3. **Ensure a session log exists** for the work in play. If none for today, create one via
   the `obsidian-session-logging` skill (invoke it — don't hand-roll frontmatter). If one
   exists, use it.
4. **Fill the `## Resume pointer` section** of that log. Machine-crisp, not prose; verify each
   line against the state captured in step 1, don't write what you *hoped* happened:
   - **Working state** — update set (name + sys_id) / git branch + HEAD + ahead-count / scratch dir.
   - **Landed** — records/files changed this session and where they went (which set/commit, pushed?).
   - **NEXT action** — the single first thing the resuming session should do.
   - **Frontier** — what's unblocked after that.
   - **Pending verification** — human F1, code review, deploy, sync — anything not yet confirmed.
5. **Write the per-id pointer** `<project>/Session Logs/_RESUME-<id>.md` (overwrite if same id)
   with the step-1 fingerprint and the pointer block. Set `status: live` if work is in flight,
   `status: complete` only if there is genuinely nothing to resume.
6. **Upsert the index row** in `<vault-root>/_CHECKPOINTS.md` (create the file with the header +
   table if missing). Newest on top.
7. **Overwrite the latest alias** `<project>/Session Logs/_RESUME.md` with a copy of this per-id
   pointer (back-compat).
8. **Fill the rest of the log** (Summary/Changes/Open Issues/Changed Files) — a light pass via
   `obsidian-session-logging` is fine; the pointer is the load-bearing part. Optionally run that
   skill's full wrap-up (daily log, standup, index) if the user wants it — ask, don't assume.
9. **Tell the user it's safe to `/clear`, and SURFACE THE ID prominently**, e.g.:
   > Checkpoint **`k7f3`** saved (EIS 2.0). Resume with **`/checkpoint-resume k7f3`**.
   Also name the session-log path + the one-line NEXT action. Stop — they press clear.

## Resume (after `/clear`, fresh session)

Two entry points. **With an id** (`/checkpoint-resume <id>`, or the user pastes one) — go
straight to it. **Without an id** — list and ask; never guess by mtime.

1. **Resolve the pointer.**
   - **Id given:** open `<vault-root>/_CHECKPOINTS.md`, find the row for that id, and read the
     `pointer` file it names. If the id isn't in the index, say so and list the live rows.
   - **No id:** read `_CHECKPOINTS.md` and show the `status: live` rows (newest first: id,
     project, checkpoint_at, NEXT). Ask the user which id to resume. Do NOT auto-pick, and do
     NOT fall back to mtime over `_RESUME.md` files.
2. **Reconcile the fingerprint against LIVE state before trusting a word of it.** Re-run the
   step-1 commands (`git branch --show-current`, `git rev-parse --short HEAD`, ahead-count,
   `git status --short`; `sndeck us get` for SN) and diff against the pointer's frontmatter.
   Then apply the **staleness guard**:
   - **Pointer missing**, OR its `status: complete`, OR live state has moved past the fingerprint
     (HEAD advanced, ahead-count grew, dirty when it claimed clean, different branch): the pointer
     is **stale or spent — do NOT narrate it as current**. Tell the user plainly ("no live
     checkpoint for that id" / "checkpoint is stale — work moved past it"), then report **actual
     current state derived from git/sndeck** and ask what to resume.
   - **Fingerprint matches live state**: proceed.
3. **Re-hydrate working state** the pointer names — confirm the update set is current, the branch
   is checked out, scratch exists. Fix drift before acting.
4. **Consume the pointer on resume.** Once picked up, flip that id's row to `status: complete` in
   `_CHECKPOINTS.md` (and the per-id file) so a spent pointer is never re-offered as live. If work
   continues and you re-checkpoint, it flips back to `live` under the SAME id (step 2 of Checkpoint).
5. **Restate** where things stand + the NEXT action in one or two lines, then continue.

If someone hand-wrote a session log but no per-id pointer / index row exists, that log was never
checkpointed — treat it as history, not a live handoff, and reconstruct current state from git/sndeck.

## Notes

- This skill is the *discipline*; `obsidian-session-logging` does the log file I/O (templates,
  frontmatter, daily log, standup, index). The per-id pointer, the `_CHECKPOINTS.md` index, and
  the `_RESUME.md` alias are owned by THIS skill — write them directly.
- `/checkpoint-resume [id]` is the dedicated resume entry point (its own skill); it defers all
  reconcile/staleness detail to the Resume section above.
- The harness auto-summarizes context across `/clear`, but that's lossy — the written pointer is
  the durable, high-fidelity handoff. The fingerprint is what keeps it honest across time.
