---
name: checkpoint
description: Use to checkpoint work before clearing context and to resume after. Trigger on /checkpoint, "checkpoint", "save progress before I clear", "wrap up so I can /clear", "resume from where we left off", "pick up the last checkpoint". Bridges the update-log -> /clear -> resume loop by writing/reading a machine-crisp Resume pointer in the session log.
---

# checkpoint

Bridges the **update session log → `/clear` → resume** loop. Two directions; pick by
what the user is doing.

The `/clear` itself is a harness command only the human runs — this skill CANNOT clear
its own context. So the checkpoint direction ends by telling the user it's safe to clear;
the resume direction starts a fresh session from the written pointer.

Companion to the `obsidian-session-logging` skill (same plugin): that skill does the
actual file I/O — templates, frontmatter, daily log, standup, index. This skill is the
checkpoint *discipline* and delegates to it rather than duplicating its commands.

## Checkpoint (before `/clear`)

Goal: leave a **Resume pointer** precise enough that a fresh session (zero memory of this
conversation) can continue without re-deriving anything.

1. **Ensure a session log exists** for the work in play. If none for today, create one via
   the `obsidian-session-logging` skill (invoke it — don't hand-roll frontmatter). If one
   exists, use it.
2. **Fill the `## Resume pointer` section** (it's in the Session Log template). Keep it
   machine-crisp, not prose — verify each line against real state, don't write what you
   *hoped* happened:
   - **Working state** — current update set (name + sys_id) / git branch / scratch dir.
   - **Landed** — records/files changed this session and where they went (which set, pushed?).
   - **NEXT action** — the single first thing the resuming session should do.
   - **Frontier** — what's unblocked after that.
   - **Pending verification** — human test, code review, deploy, sync — anything not yet confirmed.
3. **Fill the rest of the log** (Summary/Changes/Open Issues/Changed Files) — a light pass
   via `obsidian-session-logging` is fine; the Resume pointer is the load-bearing part.
   Optionally run that skill's full wrap-up (daily log, standup, index) if the user wants it
   — ask, don't assume.
4. **Tell the user it's safe to `/clear`**, and name the session-log path + the one-line
   NEXT action so they know what resuming will pick up. Stop — they press clear.

## Resume (after `/clear`, fresh session)

1. **Read the latest session log's `## Resume pointer` first** — before exploring code or
   asking questions. Find it: newest file in the relevant `<project>/Session Logs/` (or ask
   which project).
2. **Re-hydrate working state** the pointer names — confirm the update set is current, the
   branch is checked out, scratch exists. Fix drift before acting.
3. **Verify, don't trust** — the pointer reflects what was true at write time. Spot-check the
   "landed" claims before building on them.
4. **Restate** to the user where things stand + the NEXT action in one or two lines, then
   continue the work.

## Notes

- The harness auto-summarizes context across `/clear`, but that's lossy — the written
  Resume pointer is the durable, high-fidelity handoff that survives a full clear.
