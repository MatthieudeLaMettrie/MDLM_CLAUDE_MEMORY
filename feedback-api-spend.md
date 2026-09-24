---
name: feedback-api-spend
description: "Never spend Anthropic API credit (scripts using ANTHROPIC_API_KEY) without Matthieu's explicit go-ahead and a dollar estimate — after a ~$100 overrun on 2026-09-23"
metadata:
  node_type: memory
  type: feedback
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-24T04:02:38.666Z
---

**Rule:** do not run anything that calls the Anthropic API with his key
(`DataCertPrep/.env` `ANTHROPIC_API_KEY` — e.g. `scripts/generate-content.ts`,
`scripts/rebalance-questions.ts` default/batch/explanations modes) unless he
explicitly approves that specific run. Free alternatives first. DB imports,
builds, tests and in-session work don't touch that key.

**Why:** on 2026-09-23 an Opus 5.5 bulk question rewrite cost him ~$100 for
~918 questions. He said "it's costing way too much credits", then "don't use
credit" (repeated twice), then asked why it cost $100. Causes were mine:
most expensive model by default, token estimate instead of a dollar estimate,
ignored hidden thinking tokens, no usage logging or cap, discarded paid work
on interruption, 12 concurrent requests, and a stop that didn't kill child
processes.

**How to apply when a paid run is genuinely needed:**
1. Propose it with a **dollar** estimate from a tiny measured pilot (log
   `usage` incl. thinking/output tokens), cheapest adequate model first.
2. Hard budget stop in the script; log running spend.
3. Save work per request/chunk so interruptions don't waste paid calls.
4. After any stop, verify no processes remain (PowerShell by command line).
5. Suggest he sets a workspace spend limit in the Anthropic console.

Related: [[project-datacertprep-todo]], [[feedback-autonomy-and-verification-bar]].
