---
name: weekly-update
description: Use when writing the weekly update a manager reads — rolling up a week of standups and session logs into wins / follow up / still moving / note worthy. Triggers on /weekly-update, "weekly update", "update for Huston", "what did the team do this week", "manager summary of the week".
---

# Weekly Update

One week of the team's work, written for a manager who **harvests** it — he lifts lines
out of it into his own artifact covering several teams and 60–80 people.

**The audience model drives every rule below.** He is not reading to understand our
systems; he is scanning for what changed for people, alongside four other teams' updates.
A line that only makes sense next to our *other* lines is dead weight in his artifact.

Invocation: `/weekly-update [focus: <what leadership is leaning on>] [as of <date>]`

## The admission test

A line exists only if **someone outside the team can now do something they couldn't, or is
no longer affected by something they were.**

Note worthy is the one override: something the manager would be embarrassed to hear from
someone else first earns a line even when nothing changed for a user.

Everything else — our chores, our progress, our effort — has no line. This is the rule the
draft will fight you on; see Common failures.

## What each line IS

Each item is **one sentence** stating the outcome in the words a non-engineer would use. A
second sentence is allowed only when *how it changed from the previous state* is the point
("they used to be rebuilt nightly, so nothing survived 24 hours").

| Section | A line here IS |
|---|---|
| **wins — in prod** | Something a person outside the team can experience today. All real work qualifies, priority or not. |
| **follow up — in prod but still getting updates** | Live, and still changing for that person. |
| **still moving** | In flight *and* laddering to what leadership is leaning on right now (the `focus:` argument). |
| **note worthy** | Bad data or broken measurement beyond our own work; a near miss with what changed so it can't recur; a structural change in how work gets done; cross-team interaction; a risk now closed. Max 4. |

Shipped-but-not-operational is **not** a win. Code committed to production while the job,
the channel, or the permission that makes it run doesn't exist yet belongs in follow up,
saying so.

Volume ceiling ≈ 20 items (≈ 6 / 4 / 6 / 4). Enforce it by **cutting the weakest items,
not by compressing all of them.** Ranking is the work; uniform clipping is the failure.

## Voice

- **Explain what numbers mean; don't quote them.** A figure survives only when its size is
  the news for someone outside the team, or when it proves something was broken worse than
  assumed. Effort metrics — artifacts reviewed, rows verified, log entries, tests passing —
  never appear.
- **Gloss our coinages inline, every week:** VMS (the vendor questionnaire portal dealers
  use), DRMS (the Azure-side sync engine behind EIS). Company terms — EGCS, EIS — pass
  unglossed. No plumbing vocabulary at all: update set, dev, prod, ACL, business rule,
  dot-walk, GlideRecord.
- **Names sparingly**, only where the manager already associates that person with that
  work; "we" otherwise. Team name appears once in the header, never in the body. Names are
  never a scoreboard.
- **Cross-team items are facts, not asks.** "Paused pending sign-off from the cyber team"
  is status. "Waiting on Kaitlyn" is a complaint with a name on it. The manager is not a
  blocker for anyone, so nothing is framed for him to unblock.
- **Every line stands alone.** If a reader would have to ask "what does that mean?" or
  "which system?", it isn't finished. Compressing four findings into "several ways in were
  open" is the shape to avoid.

Header line: `Bear Hands (YYYY-MM-DD)` — the window's end date.

## Sourcing and verification

**Window** = seven days ending on the requested day, defaulting to the most recent Tuesday.
Work that landed just after the boundary counts and says so ("live as of this morning").

Read, in this order:
1. `Standups/` for every date in the window — what people chose to surface.
2. Every session log and daily log whose frontmatter `date` falls in the window — what
   actually happened. Session logs count even when the standup missed them.
3. Last week's `Weekly Updates/` note — for follow-through.

Team = Bear Hands: Caleb, Kelly, Bill, Zane.

**Verify every production claim against its session log before printing anything.**
Standups are written for us and overstate maturity: "promoted" in a standup routinely means
the code is there, not that anyone benefits. The session log carries the real state, the
open issues, and the manual steps still outstanding.

A claim that can't be traced to a session log gets softened or dropped — never printed and
never flagged afterwards for the user to check.

## Red flags — you are about to print something false

- "The standup says it's promoted, that's good enough."
- "The number appears twice, it must be right." (One is often a superset — 116 rows matched,
  111 actually became visible.)
- "It says the job ran, so it's live." (Ran where? Scheduled jobs and webhook rows are
  frequently uncaptured and never exist in production.)
- "I'll note the uncertainty in a caveat under the draft."

All of these mean: open the session log named in the standup and read its Open Issues.

## Follow-through

An item that was in flight last week and hasn't moved is carried **once**, marked plainly
as unchanged, then disappears until something happens to it. An item reappearing unchanged
every week trains the reader to skim the section it's in.

## Output

1. The update, in chat, ready to send.
2. **A cut list** — what was left out and why, in one line each. This is where the judgment
   is visible and where the user catches a wrong cut.
3. The note written to `Weekly Updates/YYYY-MM-DD.md` via the create recipe with
   `sn_category=weekly_update` (see the `obsession` skill for the recipe and CLI usage).

If `focus:` wasn't supplied, ask once — one question, then proceed. Never store the focus;
leadership priorities drift and a stale note silently weights the wrong section for weeks.

## Common failures

These are observed, not hypothetical — every one of them shipped in a draft.

| Failure | Looks like | Fix |
|---|---|---|
| Status reporting | "VMS usage numbers get more complete over the next few weeks." | Nobody's day changed. Cut it. |
| Our chores as news | "Re-measuring on production this week." / "Dev is broken, production is correct." | Internal work. Cut it. |
| Vague compression | "Several ways in were open and are now closed." | Name the one that matters, in plain words. |
| Dangling reference | "…behind the sync rebuild." | Say what it is, or don't mention it. |
| Trusting the standup | Calling a feature live that has no channel and no job in production. | Read the session log's Open Issues. |
| Effort as impact | "448 artifacts reviewed, 992 rows verified." | Say what it protected the customer from. |
| Confusing framing | "Five items waiting on a person, not on engineering." | Just name the items and their state. |
