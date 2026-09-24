# 2026-09-23 → 24 — Content-accuracy fixes, answer-bias rework, cost incident

Continuation of [2026-09-22 session](2026-09-22-datacertprep-redesign-and-curriculum-audit.md).
Repo: `C:\Users\mettrma\Documents\GitHub\DataCertPrep` (github.com/MatthieudeLaMettrie/DataCertPrep).
Open work is tracked in [[project-datacertprep-todo]].

## Timeline

| When | What | Where |
|---|---|---|
| 09-23 | Set up this memory repo (`MDLM_CLAUDE_MEMORY`, private) | this repo |
| 09-23 | Logged external Codex "ASTRA-6" audit findings | [[project-datacertprep-content-audit-2026-09-23]] |
| 09-23 | Fixed confirmed factual errors + SEO issues from that audit | PR #47 (merged 09-23) |
| 09-23 | Added Vercel Speed Insights | PR #48 (merged 09-23) |
| 09-23 | Measured answer bias bank-wide; built rebalance tooling; Opus rewrite (partial, stopped for cost); free shuffle | PR #49 (merged 09-23), imported to prod |
| 09-24 | A–D labels in practice/exam UI; free letter-remap shuffle | PR #50 (merged 09-24), imported to prod |

## 1. ASTRA-6 audit fixes (PR #47)

**What:** Matthieu pasted an independent Codex audit. He chose to fix: confirmed
factual errors, SEO/structured-data issues, the AI-103 article. Not: answer bias
(done later, below).

**How:** each claim verified against a primary source before editing (the
standing evidence bar in [[feedback-autonomy-and-verification-bar]]):
- AWS DEA-C01 guide — KDS standard read throughput is *shared* across consumers
  (guide contradicted its own cheat sheet); Firehose supports a **0 s** buffer
  interval (AWS Firehose docs table), not a 60 s minimum. Fixed guide text, Q&A,
  limits table, one question explanation.
- Microsoft DP-700 guides — Microsoft publishes all three domains as **30–35%**
  ranges (live Learn study guide), so no domain is "the largest" (all three
  guides claimed it). KQL *does* support `join` (the guide's own example used one).
- GH-300 — privacy/safeguards guide still had old title "PRs, Projects, Admin"
  and 20% weight; set to current name and 14%.
- dbt — question claimed a `varchar(50)` vs `varchar(100)` mismatch fails a
  contract; dbt docs say contracts don't compare size/precision/scale. Answer
  key fixed, guide sentence fixed.
- AI-103 article (`lib/articles.tsx`) — removed "shifts entirely to generative
  AI"; vision/text/extraction domains still exist.
- SEO — removed login-gated `/practice`, `/flashcards`, `/login` from
  `app/sitemap.ts`; Quiz JSON-LD now lists *all* correct answers on
  multi-answer questions; removed FAQPage JSON-LD (Google dropped FAQ rich
  results May 2026) — visible FAQ UI kept.

## 2. Speed Insights (PR #48)

`npm i @vercel/speed-insights`, `<SpeedInsights />` in `app/layout.tsx`.
Matthieu enabled it in the Vercel dashboard.

## 3. Answer bias (PRs #49, #50)

**Finding** (`scripts/audit-answer-bias.ts`, 9,343 single-answer questions):
correct answer was the **longest option 70%** of the time and sat at
**A35/B47/C15/D2**; ten certs had *every* answer at A. Cause: generator prompt
never constrained length/position, and the app shuffles questions but never
options.

**What was built:**
- `lib/question-balance.ts` (+ tests): seeded per-stem option shuffle;
  letter-reference detection; `shuffleWithLetterRemap` (rewrites unambiguous
  "Option B"/"(C)"/"The correct answer is B" letters to the new order).
- `scripts/generate-content.ts`: prompt now demands length parity and
  letter-free explanations, then shuffles — new content won't be biased.
- `scripts/rebalance-questions.ts` modes:
  - default / `--batch-submit|--batch-collect`: Opus rewrite of distractors +
    explanations (costs API credit).
  - `--shuffle-only`: free, shuffles letter-free questions.
  - `--remap-letters`: free, shuffles + rewrites unambiguous letter refs.
  - `--explanations-only`: Haiku, never run.
- Practice + exam UI now show A–D labels (they had none, though ~6,700
  explanations say "Option C…").

**Results, live in prod:**
- ~918 questions fully rewritten by Opus (all of databricks-ml-professional,
  parts of ai-advanced / ai-foundations / ai-intermediate). Pilot cert:
  longest-correct 97% → 33%.
- 3,368 free-shuffled + 4,767 letter-remapped.
- Bank-wide position now **A26/B29/C23/D22**. Longest-correct still **~66%**.
- Every change verified: options, correct answer and letter→option mapping
  unchanged (0 real mismatches); prod read-back 11,013/11,013 match.
- The rewrite model flagged **139 real accuracy concerns** →
  `docs/audits/question-rebalance-concerns.md` (not changed; needs human review).

**How it shipped:** content files → PR → merge → `import-content.ts <cert>
--publish` for all 47 certs (upsert by stable id = sha1(cert, domain, stem);
no `--prune`) → read-back verify script.

## 4. Cost incident — ~$100 of API credit (09-23)

The Opus 5.5 rewrite runs cost Matthieu about $100. Causes (my errors): picked
the most expensive model by default for a bulk output-heavy job; gave a token
estimate not a dollar estimate and ignored hidden thinking tokens; no usage
logging or spend cap; paid work discarded when credit ran out / runs stopped;
12 parallel requests; stopping the background job didn't kill its node
children, which ran ~1 min more. A Message Batch was tried (half price) but
sat at 0/1,506 for 9 h and was cancelled at no cost. Matthieu then said
**"don't use credit"** — rule recorded in [[feedback-api-spend]].

## 5. Machine problems hit

- Windows started blocking `esbuild.exe` (`spawn EPERM`) mid-session →
  `npx tsx` silently does nothing. Workaround + details in
  [[reference-datacertprep-infra]].
- Stopping an `xargs -P` background job leaves node children running on
  Windows — kill by command line via PowerShell.

## Other facts learned

- GH-600 exists on the site as `github-agentic-ai-developer` (6 domains,
  verified 2026-08-10, thin: 46 single-answer questions).
- MLA-C01 retirement banner was already implemented; flips to "retired" on
  2026-09-29 automatically.
- DP-700 and PL-300 each have 1 stale duplicate published question.
