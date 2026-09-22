---
name: feedback-autonomy-and-verification-bar
description: "When to proceed autonomously vs. pause, and how much evidence is enough before editing live data"
metadata:
  node_type: memory
  type: feedback
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-22T23:17:17.188Z
---

Confirmed over a full day of production work on DataCertPrep (2026-09-22/23,
the redesign + curriculum-audit + content-generation session).

**Proceed decisively once direction is given.** He's given approvals like
"go ahead", "proceed", "yes, go ahead" for large multi-step chains (generate
content → spot-check → publish to prod; merge a PR; rotate + re-sync a DB
password across Neon/Vercel) and expects the whole chain executed without
re-confirming each step, unless a new decision point comes up that he
genuinely hasn't weighed in on yet (e.g. a monetization/product call like
whether to pull a paid report for a retired exam — for that I used
AskUserQuestion rather than guessing, and he answered directly. That was the
right call: he didn't push back or say "just decide it yourself").

**Where he still wants a pause**: this session's own auto-mode classifier
blocked direct production DB writes and required him to Shift+Tab out of
auto mode each time (documented as a known, expected gotcha in his own
CLAUDE.md for this repo). I should still attempt the action, hit the block,
explain plainly what's needed, and wait — not try to route around it.

**Calibrated evidence bar for editing live/published data — confirmed
correct behavior, don't loosen it.** When investigating whether certification
exam blueprints had drifted from official vendor pages, I only edited data
when I had either (a) a real primary-source fetch, or (b) at least two
independent secondary sources converging on the exact same numbers. When
evidence was single-source, stale, or self-contradictory (e.g. two fetches
of the same AWS PDF returning different, mutually exclusive domain
structures), I explicitly left the existing data unchanged and said why,
rather than guessing. He never pushed back on this caution or asked me to
lower the bar — if anything he kept asking me to dig further for better
evidence rather than accepting "unconfirmed" as a final answer. Keep this
same discipline on any future task involving correcting live content against
external sources: more digging is welcome, guessing on thin evidence is not.

**Security hygiene he cares about but doesn't always follow himself.** He
has a standing TODO (in the DataCertPrep repo's CLAUDE.md) to rotate a Neon
DB password because it was pasted into a chat session once before. Mid-session
he pasted the live prod Neon password into chat again despite my suggesting
a safer `!`-prefixed file-write pattern first. I flagged the repeat exposure
plainly (not preachy) and it led to him actually rotating the password later
in the same session. Keep suggesting the safer pattern proactively for any
credential, but don't block on it if he pastes it anyway — handle it,
don't echo it back, and note the exposure once.
