---
name: checkpoint
description: Use to checkpoint work before clearing context and to resume after. Trigger on /checkpoint, "checkpoint", "save progress before I clear", "wrap up so I can /clear", "resume from where we left off", "pick up the last checkpoint". Bridges the update-log -> /clear -> resume loop by writing/reading a machine-crisp Resume pointer plus a deterministic _RESUME.md pointer file.
---

# checkpoint

Bridges the **update session log → `/clear` → resume** loop. Two directions; pick by
what the user is doing.

The `/clear` itself is a harness command only the human runs — this skill CANNOT clear
its own context. So the checkpoint direction ends by telling the user it's safe to clear;
the resume direction starts a fresh session from the written pointer.

## The deterministic pointer: `_RESUME.md`

Resume must NEVER guess which log is current by file mtime — that silently picks a stale,
already-`complete` log when no fresh checkpoint was written (the exact failure this skill
exists to prevent). Instead, every checkpoint writes ONE well-known file per project:

`<project>/Session Logs/_RESUME.md`

It is overwritten on every checkpoint and is the single source of truth for "what to resume
from." It carries a live-state fingerprint so a fresh session can instantly tell whether it's
still current or stale. Schema:

```markdown
---
type: resume-pointer
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

## Checkpoint (before `/clear`)

Goal: leave a pointer precise enough that a fresh session (zero memory of this conversation)
can continue without re-deriving anything.

1. **Capture live state FIRST, before writing anything.** Run the real commands so the
   fingerprint is true, not remembered: `git branch --show-current`, `git rev-parse --short HEAD`,
   `git status --short`, ahead-count (`git rev-list --count main..HEAD` or vs. the PR base),
   and — for SN work — `sndeck us get`. You cannot write an honest pointer from memory.
2. **Ensure a session log exists** for the work in play. If none for today, create one via
   the `obsidian-session-logging` skill (invoke it — don't hand-roll frontmatter). If one
   exists, use it.
3. **Fill the `## Resume pointer` section** of that log. Machine-crisp, not prose; verify each
   line against the state captured in step 1, don't write what you *hoped* happened:
   - **Working state** — update set (name + sys_id) / git branch + HEAD + ahead-count / scratch dir.
   - **Landed** — records/files changed this session and where they went (which set/commit, pushed?).
   - **NEXT action** — the single first thing the resuming session should do.
   - **Frontier** — what's unblocked after that.
   - **Pending verification** — human F1, code review, deploy, sync — anything not yet confirmed.
4. **Write `_RESUME.md`** (overwrite) with the step-1 fingerprint and the pointer block. Set
   `status: live` if work is in flight, `status: complete` only if there is genuinely nothing to
   resume. This file — not mtime — is what resume reads.
5. **Fill the rest of the log** (Summary/Changes/Open Issues/Changed Files) — a light pass via
   `obsidian-session-logging` is fine; the pointer is the load-bearing part. Optionally run that
   skill's full wrap-up (daily log, standup, index) if the user wants it — ask, don't assume.
6. **Tell the user it's safe to `/clear`**, and name the session-log path + the one-line NEXT
   action. Stop — they press clear.

## Resume (after `/clear`, fresh session)

1. **Read `<project>/Session Logs/_RESUME.md` first** — the deterministic pointer. (Ask which
   project only if ambiguous.) Do NOT sort logs by mtime to pick a resume source.
2. **Reconcile the fingerprint against LIVE state before trusting a word of it.** Re-run the
   step-1 commands (`git branch --show-current`, `git rev-parse --short HEAD`, ahead-count,
   `git status --short`; `sndeck us get` for SN) and diff against the pointer's frontmatter.
   Then apply the **staleness guard**:
   - **`_RESUME.md` missing**, OR its `status: complete`, OR the live state has moved past the
     fingerprint (HEAD advanced, ahead-count grew, dirty when it claimed clean, different branch):
     the pointer is **stale or spent — do NOT narrate it as current**. Tell the user plainly
     ("no live checkpoint" / "checkpoint is stale — work moved past it"), then report **actual
     current state derived from git/sndeck** and ask what to resume. Never present a stale log
     as if it were the live handoff.
   - **Fingerprint matches live state**: proceed.
3. **Re-hydrate working state** the pointer names — confirm the update set is current, the branch
   is checked out, scratch exists. Fix drift before acting.
4. **Restate** where things stand + the NEXT action in one or two lines, then continue.

If someone hand-wrote a session log but no `_RESUME.md` exists, that log was never checkpointed —
treat it as history, not a live handoff, and reconstruct current state from git/sndeck.

## Notes

- This skill is the *discipline*; `obsidian-session-logging` does the log file I/O (templates,
  frontmatter, daily log, standup, index). `_RESUME.md` is owned by THIS skill — write it directly.
- The harness auto-summarizes context across `/clear`, but that's lossy — the written pointer is
  the durable, high-fidelity handoff. The fingerprint is what keeps it honest across time.
