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
checkpoint_at: <YYYY-MM-DD HH:MM ±HHMM>  # when this pointer was written — INCLUDE THE OFFSET.
                                          # Resume compares this against `sys_updated_on`,
                                          # which is instance time (commonly UTC). A bare
                                          # local timestamp compared to a UTC one is off by
                                          # hours and will misjudge staleness at the boundary.
session_log: "[[<full session-log note title>]]"
status: live | complete | stale           # live = work in flight; complete = resumed or nothing to
                                          # resume; stale = abandoned, do not offer as resumable
# --- working-state fingerprint ---
# Fill the block for each surface in play (step 1). Omit blocks for surfaces
# that aren't — an all-n/a block is noise, and a MISSING block is a claim you
# never checked. If NO surface applies, write `surfaces: none` and say what
# does pin the work (vault files, a scratch dir, an external ticket).
surfaces: <git | servicenow | git+servicenow | none>
# -- git (when in play) --
git_repo: <path>                          # branch + sha mean nothing without it, and one
                                          # vault project can span several repos
git_branch: <branch>
git_head: <short sha>
git_ahead_of_base: <N vs. the base named below>
git_base: <branch or ref the ahead-count is measured against>   # on the default branch
                                          # with uncommitted work, that's `origin/<branch>`
git_dirty: <true/false>
git_dirty_files:                          # WHICH files, not just whether. A bare boolean
  - <path>                                # cannot catch "different files dirty now" — the
                                          # git counterpart of records_in_flight. Omit if clean.
# -- servicenow (when in play) --
instance: <dev | prod | other name>       # REQUIRED for SN. A pointer without it
                                          # reconciles clean against the wrong instance.
update_set: "<name> (<sys_id>)"
update_set_state: <in progress | complete | ignore>   # lowercase, normalised — the tools
                                          # disagree (sndeck us get says "In progress",
                                          # MCP says "in progress", sndeck status prints
                                          # "complete"/"ignore"). Resume string-matches
                                          # this, so normalise on write.
sndeck_staged: <none | N staged: X local-only, Y dirty, Z clean>
  # `local-only` is its own value and MUST NOT be folded into "clean".
  # sndeck's summary line "clean — nothing to push" means "no diff vs. the
  # pull-time snapshot" — NOT "pushed". A record can be local-only and clean
  # at once. Recording that as "clean" asserts work landed that never did.
records_in_flight:                        # ONLY records this session actually worked on,
  - <table> <sys_id> @ <sys_updated_on>   # with their instance timestamp at checkpoint time.
                                          # This is what makes "someone else changed it
                                          # since" detectable at all.
  # Derive candidates from live state (step 1), then NARROW to this session's work.
  # A sndeck staging area accumulates for months and across users — listing all of it
  # here inverts the field into a false-alarm generator, because resume fires staleness
  # the moment ANYONE touches ANY listed record. Everything staged but inert goes below.
staged_but_inert:                         # staged, not this session's work. Recorded so
  - <table> <sys_id>                      # resume knows they were staged without arming
                                          # a staleness signal on them. Omit if none.
other_sets_dirty:                         # sndeck status warns about unpushed edits
  - "<set name> [<state>] — <table>/<record>"   # stranded in OTHER sets. Often the most
                                          # actionable thing on screen. Omit if none.
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

`status` is `live` | `complete` | `stale`. Only `live` rows are offered by a bare
`/checkpoint-resume`. `stale` exists so an abandoned thread can stop being offered without
being falsely recorded as finished — see Resume step 4.

## Checkpoint (before `/clear`)

Goal: leave a pointer precise enough that a fresh session (zero memory of this conversation)
can continue without re-deriving anything.

1. **Capture live state FIRST, before writing anything.** You cannot write an honest pointer
   from memory — run the real commands.

   **Name the surfaces before running anything.** Which of these does this work actually live
   on: a git repo, a ServiceNow instance, both, or neither? Run only the row(s) that apply.
   Running the other paradigm's commands produces a screen of errors and teaches you nothing.

   **Two surfaces in one session is normal — run both rows and record both.** A finding on one
   surface is never a reason to skip the other; the ServiceNow rows below carry more prose only
   because they have more failure modes, not because they matter more. (If the two surfaces are
   genuinely unrelated workstreams that merely overlapped in time, that is two threads — give
   them two ids and two pointers.)

   | Surface | Capture with | Fills |
   |---|---|---|
   | **git** | `git rev-parse --show-toplevel` · `git branch --show-current` · `git rev-parse --short HEAD` · `git status --short` (record the paths, not just the fact) · `git rev-list --count <base>..HEAD` (name the base you used — `main` is a guess, not a default) | `git_repo` `git_branch` `git_head` `git_dirty` `git_dirty_files` `git_ahead_of_base` `git_base` |
   | **ServiceNow** | `sndeck us get --json` (set name, sys_id, **state**, and the `sndeck: instance <name>` banner on stderr) · `sndeck status` (staged records + per-record push state + warnings about other sets) | `instance` `update_set` `update_set_state` `sndeck_staged` `other_sets_dirty` |
   | **ServiceNow, records in flight** | **Derive the record list from live state, not memory:** `sndeck status` staging area + `SN-Inspect-Update-Set` components. Then read each one's `sys_updated_on` (MCP `SN-Query-Table`/`SN-Get-Record`) | `records_in_flight` |
   | **neither** | — | `surfaces: none`, plus a plain-language note on what *does* pin the work |

   **`sndeck us get` alone is not enough.** It reports which set is current; it says nothing
   about whether your work is pushed. `sndeck status` is the true counterpart of
   `git status --short` — it is what distinguishes "I pushed it" from "it is sitting local-only
   in a scratch dir." A pointer claiming work landed, written without running it, is fiction.

   **Capture the instance, and cross-check both pointers.** Every sndeck invocation prints
   `sndeck: instance <name>`; the MCP keeps its **own independent** instance pointer
   (`SN-Get-Current-Instance`). Confirm they agree before recording. Without the field a pointer
   reconciles as a clean match against the *wrong instance*; with the field but no cross-check,
   every `sys_updated_on` you record can come from a different instance than the set you name.
   On a human-gated prod that is the most expensive failure this skill can cause.

   **Reconcile live state against what you believe you did — before writing anything.** The
   resume side has a whole staleness apparatus for "has the world moved?"; the capture side needs
   the mirror of it. If the instance contradicts your memory of the session — the current set holds
   none of the records you think you pushed, nothing was updated today, the set's contents predate
   your session — then **stop and reconcile with the user.** Do not write the pointer. A confident
   `records_in_flight` list assembled from memory, against an instance that disagrees, is precisely
   the fiction this step exists to prevent. Most often it means your work went into a different
   update set than the one that is current.

   **Check the current set actually belongs to this work.** The pointer's project and the current
   set's name should be about the same thing. A "Widget Project" checkpoint whose current set is
   "EIS 2.0 SyncOrchestrator M1" is a signal that either the set pointer drifted mid-session (it is
   per-user and shared) or the pushes landed somewhere unintended. Resolve it before writing, not
   on resume.

   **Note the vault project too.** The pointer's own path is `<project>/Session Logs/_RESUME-<id>.md`,
   and nothing in git or sndeck output names a vault project. Get it from the session log or ask —
   inferring it from an update set's name is a guess.
2. **Determine the id.** Reuse the id this session was resumed from, if any; otherwise mint a
   new 4-char id and confirm it's absent from `_CHECKPOINTS.md`.
3. **Ensure a session log exists** for the work in play. If none for today, create one via
   the `obsession` skill (invoke it — don't hand-roll frontmatter). If one
   exists, use it.
4. **Fill the `## Resume pointer` section** of that log. Machine-crisp, not prose; verify each
   line against the state captured in step 1, don't write what you *hoped* happened:
   - **Working state** — whatever step 1's surface table filled, in prose. For SN: instance,
     update set (name + sys_id + state), push state, scratch dir. For git: branch + HEAD +
     ahead-count vs. its named base. Say which surfaces are NOT in play, so the resuming
     session doesn't hunt for a repo that doesn't exist.
   - **Landed** — records/files changed this session and where they went (which set/commit, pushed?).
   - **NEXT action** — the single first thing the resuming session should do.
   - **Frontier** — what's unblocked after that.
   - **Pending verification** — human F1, code review, deploy, sync — anything not yet confirmed.

   This section is a **convenience copy**. The per-id pointer and the `_CHECKPOINTS.md` row are
   authoritative for state; when they disagree with the log, the registry wins. Do not add durable
   copies of state anywhere else in the log — narrative goes in step 5's block, state stops here.
5. **Append the checkpoint block to the session log — the landed delta ONLY.** State goes stale;
   narrative does not. "The funnel shipped as a second pass on 2026-08-25" is true forever; "the
   current update set is X" was true for about an hour. The pointer carries state and is *consumed*
   on resume; the session log accumulates the story, so a thread that survives three `/clear`s can
   still be written up at wrap-up by an agent that remembers only the last segment.

   Append under a `## Checkpoints` section — a genuinely new section, so the CLI's `append` is the
   right tool here (see `obsession` → *Writing to session logs*):

   ```markdown
   ## Checkpoints

   ### `<id>` · <YYYY-MM-DD HH:MM>

   - <what landed since the previous block — one bullet per outcome>
   ```

   The id in the heading is what ties this narrative to its registry row and its pointer file. One
   thread keeps one id for its whole life, so the same id recurs and the **timestamp** is what
   separates the blocks.

   **Landed delta only — nothing else.** Not NEXT, not the current set/branch/instance, not the
   frontier, not pending verification. Those are state; they live in the pointer, single-sourced,
   and resume consumes them. Every extra durable copy is one more thing that can lie to a future
   session: a Resume pointer went stale twice inside ninety minutes in one real session (the set
   was completed, then a new one created), and a copy of it pasted into the log would have outlived
   the thread with nothing to correct it.

   **Nothing landed since the last block? Write nothing.** A checkpoint taken purely to survive a
   `/clear` mid-thought has no narrative in it. An empty block is not neutral — at wrap-up it reads
   as a segment that produced nothing rather than one that never finished, and curation has to
   litigate the difference. Tell the user you skipped it instead.

   `append` writes at end of file, so blocks stay contiguous under `## Checkpoints` only while
   nothing else is appended between checkpoints. If another section landed after it, place the block
   under the existing heading with Read + Edit rather than appending a second `## Checkpoints`.

6. **Write the per-id pointer** `<project>/Session Logs/_RESUME-<id>.md` (overwrite if same id)
   with the step-1 fingerprint and the pointer block. Set `status: live` if work is in flight,
   `status: complete` only if there is genuinely nothing to resume.
7. **Upsert the index row** in `<vault-root>/_CHECKPOINTS.md` (create the file with the header +
   table if missing). Newest on top.
8. **Overwrite the latest alias** `<project>/Session Logs/_RESUME.md` with a copy of this per-id
   pointer (back-compat).
9. **Leave Summary/Changes for wrap-up.** Step 5's block is this segment's narrative record —
   don't also pre-write Summary from it, because wrap-up consolidates every block into that section
   and a half-written Summary just gives curation something to reconcile. Open Issues and Changed
   Files are fine to keep current as you go. Optionally run `obsession`'s full wrap-up
   (daily log, standup, index) if the user wants it — ask, don't assume.
10. **Tell the user it's safe to `/clear`, and SURFACE THE ID prominently**, e.g.:
    > Checkpoint **`k7f3`** saved (EIS 2.0). Resume with **`/checkpoint-resume k7f3`**.
    Also name the session-log path + the one-line NEXT action. Stop — they press clear.

### What checkpoint does NOT write

The session log is the **only** log a checkpoint touches. Not the daily log, not the standup — and
not to save effort. Each would be actively wrong:

- **Daily log** — a *verified*, cross-project roll-up written once at wrap-up, after the
  verification pass. A mid-session checkpoint has nothing verified in it yet; appending would inject
  in-flight state into the file other projects read for status.
- **Any checkpoint can end `stale`.** Work gets abandoned. Auto-appending publishes work that never
  happened into the files a reader trusts most, and no later step retracts it. The session log
  tolerates this because wrap-up curates it; the daily log has no such pass.
- **Standup** — curated and team-facing, sourced deliberately from finished session logs. It is not
  a log sink.

If a checkpoint really is the end of the day's work, it is a wrap-up, not a checkpoint — run
`obsession`'s wrap-up (step 9) and let the verification steps do their job.

## Resume (after `/clear`, fresh session)

Two entry points. **With an id** (`/checkpoint-resume <id>`, or the user pastes one) — go
straight to it. **Without an id** — list and ask; never guess by mtime.

1. **Resolve the pointer.**
   - **Id given:** open `<vault-root>/_CHECKPOINTS.md`, find the row for that id, and read the
     `pointer` file it names. If the id isn't in the index, say so and list the live rows.
   - **No id:** read `_CHECKPOINTS.md` and show the `status: live` rows (newest first: id,
     project, checkpoint_at, NEXT). Ask the user which id to resume. Do NOT auto-pick, and do
     NOT fall back to mtime over `_RESUME.md` files.
2. **Reconcile the fingerprint against LIVE state before trusting a word of it.** Re-run step 1's
   capture for the surfaces the pointer's `surfaces:` field names, and diff against its frontmatter.

   **The governing rule is: has the world moved past this pointer?** The signals below are how
   that shows up per surface — they are the known instances of the rule, not its definition. A
   surface not listed here still obeys the rule; work out its equivalent of "someone else changed
   this while I was gone."

   | Rule | git | ServiceNow |
   |---|---|---|
   | Someone else advanced the work | `HEAD` advanced | a **`records_in_flight`** record's `sys_updated_on` is **later than `checkpoint_at`** (normalise the clocks first — one is instance time, the other carries an offset). `staged_but_inert` records do **not** arm this signal |
   | More landed than the pointer knows | ahead-count grew | the set holds `sys_update_xml` rows the pointer doesn't account for |
   | Uncommitted work changed | dirty when it claimed clean, **or a different set of `git_dirty_files`** | `sndeck status` shows unpushed edits the pointer didn't record |
   | The target is gone or closed | branch deleted or merged | **`update_set_state` is now `Complete` or `Ignored` — you cannot push to it** |
   | You're aimed somewhere else entirely | different branch | **`instance` differs from the pointer's** |
   | Your local working copy is gone | worktree/checkout deleted | pointer recorded staged records, but `sndeck status` now shows **none** — the scratch was pruned and **unpushed local work is destroyed**. (Pointer recorded `sndeck_staged: none` and there is still none? Nothing was lost — not a signal.) |

   Then apply the **staleness guard**:
   - **Pointer missing**, OR its `status` is `complete` or `stale`, OR any signal above fires:
     the pointer is **stale or spent — do NOT narrate it as current**. Tell the user plainly
     ("no live checkpoint for that id" / "checkpoint is stale — work moved past it"), then report
     **actual current state derived from the live surfaces** and ask what to resume. Offer to mark
     it `stale` (step 4) so it stops being re-offered.
   - **Fingerprint matches live state**: proceed.

   **A signal you could not evaluate is not a signal that passed.** If you lacked the access or the
   data to check one (you couldn't reach the instance, the pointer never recorded
   `records_in_flight`, you can't enumerate the set's updates), the fingerprint is **unverified** —
   say which check you couldn't run and let the user decide. Never report a clean match on evidence
   you didn't gather. The same rule the schema states for capture — a missing block is a claim you
   never checked — holds on resume.

   **Precedence over step 3 — the one ambiguity that bites.** Step 3 repairs drift; step 2 decides
   whether there is anything worth repairing. **Step 3 runs only after step 2 clears.** Do not
   "fix" a fired staleness signal by quietly setting the update set and carrying on.

   The distinction that matters, because these look identical at a glance:
   - **Your pointer moved → re-hydrate (step 3).** The current update set differs from the
     pointer's, but that set is still `In progress` and nothing else changed. This is *routine* —
     the sndeck current-set pointer is **per-user and shared across sessions**, so it drifts on its
     own. Re-set it and continue; this alone is not staleness.
   - **The world moved → stale (step 2).** The set is `Complete`/`Ignored`, or a record changed
     after `checkpoint_at`, or the instance differs, or unrecorded updates landed in the set.
     Stop and ask.

   Treating the first as staleness blocks normal work; treating the second as drift silently
   resumes on top of someone else's changes.
3. **Re-hydrate working state** the pointer names — confirm the update set is current and still
   open, the branch is checked out, the staging area holds what the pointer recorded. Fix drift
   before acting.
4. **Resolve the pointer's status.** One of:
   - **Resumed** → flip that id's row to `status: complete` in `_CHECKPOINTS.md` (and the per-id
     file) so a spent pointer is never re-offered as live. If work continues and you re-checkpoint,
     it flips back to `live` under the SAME id (step 2 of Checkpoint).
   - **Stale and abandoned** → flip it to `status: stale`, with the user's OK. This is why the
     third status exists: leaving it `live` re-offers dead work on every future bare
     `/checkpoint-resume`, and marking it `complete` claims work finished that never did.
   - **Stale but the user still wants it** → leave it `live` and re-checkpoint under the same id
     once you've re-established real working state.
5. **Restate** where things stand + the NEXT action in one or two lines, then continue.

If someone hand-wrote a session log but no per-id pointer / index row exists, that log was never
checkpointed — treat it as history, not a live handoff, and reconstruct current state from the
live surfaces (step 1's table).

## Notes

- This skill is the *discipline*; `obsession` does the log file I/O (templates,
  frontmatter, daily log, standup, index). The per-id pointer, the `_CHECKPOINTS.md` index, and
  the `_RESUME.md` alias are owned by THIS skill — write them directly.
- `/checkpoint-resume [id]` is the dedicated resume entry point (its own skill); it defers all
  reconcile/staleness detail to the Resume section above.
- The harness auto-summarizes context across `/clear`, but that's lossy — the written pointer is
  the durable, high-fidelity handoff. The fingerprint is what keeps it honest across time.
