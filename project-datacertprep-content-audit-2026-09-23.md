---
name: project-datacertprep-content-audit-2026-09-23
description: External Codex/ASTRA-6 content-accuracy audit findings for DataCertPrep (2026-09-23) — specific factual errors and structural issues to fix
metadata:
  node_type: memory
  type: project
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-22T23:42:55.509Z
---

Matthieu ran an independent Codex-based review ("ASTRA 6") of DataCertPrep
on 2026-09-23, separate from the Claude Code curriculum-currency audit done
earlier the same day (see [[project-datacertprep-overview]]). It sampled 47
curriculum definitions, 223 domains' file coverage, several guides, 5
question files (69 questions), site responses, and SEO — explicitly a
**sampled** accuracy review, not full verification. Overall verdict: broad
coverage and structure are strong, but content isn't yet consistently
accurate or fully exam-aligned — priority should be fixing/verifying
existing material before adding more certs.

**Why this matters**: this is a second, independent data point beyond my own
audit — it caught real factual errors my blueprint-level check wouldn't
have (my audit checked domain/weight structure against vendor pages; this
one sampled actual guide/question *content* for correctness). Treat both as
complementary, not redundant.

**Specific confirmed issues to fix** (each has a claimed source in the
original report — verify against primary sources per the usual
[[feedback-autonomy-and-verification-bar]] discipline before editing):
- **Answer-length/position bias in questions**: in the 58-question single-answer
  sample, the correct answer was the *longest* option 69% of the time (40/58),
  and answer "B" was correct in 32/58. This is a real, fixable pattern —
  distractors need rebalancing so answer length/position stop being cues.
  Broader than any one cert; likely systemic in how questions were generated.
- **AWS DEA-C01** ingestion guide: says Kinesis standard-throughput reads are
  "per consumer" (actually shared across consumers) and claims a blanket
  60-second Firehose buffering minimum (some Firehose destinations allow
  zero buffering).
- **Microsoft DP-700** ingestion guide: says KQL can't do relational joins
  (it can), and calls its 33%-weighted domain "the largest" when another
  domain is weighted 34% — also, Microsoft publishes 30–35% *ranges*, not
  exact point allocations like 33%/34%.
- **GitHub Copilot GH-300**: the guide mapped to the privacy/safeguards
  domain still carries an old title ("PR/project administration") and old
  20% weight from before the blueprint was updated — blueprint metadata was
  updated but the underlying lesson content wasn't.
- **dbt Analytics Engineering**: a governance question (#4) claims a
  varchar(50) vs varchar(100) mismatch necessarily fails a dbt contract —
  dbt's own docs say contract validation does NOT compare type size/
  precision/scale. This is a wrong-answer-key bug, not just a wording issue.
- **Databricks DE Associate**: called out as one of the *stronger* structural
  matches (all 7 domains + weights match official page) — no action needed
  there, just noted as a positive control.
- **SnowPro Core / Terraform 004**: partial verification only; repo already
  labels some weights/logistics as estimated — audit says keep that
  labeling, don't claim exact exam simulation.
- **AI-103 article** (editorial, not guide content): claims the exam moved
  *entirely* to generative AI; Microsoft's real syllabus retains vision,
  text analysis, and information extraction too. Flagged as a trust/
  editorial-standards issue, not just a factual one — recommends named
  reviewers + primary-source citations + fact-checking before publishing
  articles going forward.

**SEO/structural issues found**:
- Sitemap includes 47 practice-question URLs and 47 flashcard URLs that
  require login, plus the login page itself — should be removed from the
  sitemap (crawl budget / low-value indexing).
- Even the nominally "free" guide requires authentication — consider
  exposing real public guide previews.
- Question JSON-LD structured data only encodes the *first* correct answer
  even on multi-answer questions — should reflect all correct answers.
- FAQ JSON-LD is now pointless: Google discontinued FAQ rich results in
  May 2026 — this markup should probably be dropped, not just left in.
- Confirmed as already fixed/deployed and NOT to re-flag: practice-score
  wording fixes from an earlier session are live; audit explicitly excluded
  stale crawl findings related to that.

**Two time-sensitive curriculum checkpoints** (reinforces, doesn't replace,
GitHub issue #46):
- AWS **MLA-C01** English version closes 2026-09-28; **MLA-C02** beta
  delivery starts 2026-09-29 — the live MLA-C01 content will need a
  retirement-style treatment soon (see the aws-mls-c01 precedent already
  shipped in `app/certs/[slug]/report/page.tsx`), and MLA-C02 will
  eventually need net-new coverage.
- Microsoft's **DP-700** page already shows objectives effective
  2026-10-19 — matches the existing issue #46 Fabric recheck date; audit
  flags that the *current* DP-700 guide content should be clearly
  distinguished from the incoming syllabus, not just rechecked later.

**Recommended fix order from the audit** (not yet actioned, pending a future
session): (1) accuracy/alignment corrections above + map each official
objective to a reviewed lesson/question with source links and review dates,
(2) assessment quality — remove answer-length/position clues, improve
distractors, validate with held-out questions, (3) SEO/trust — clean
sitemap, add guide previews, fix structured data, slow down article
publishing until sourcing/review process exists.

**Not yet done — explicitly deferred by Matthieu to a future session** (as
of 2026-09-23, same conversation this was logged in): creating a global
`~/.claude/CLAUDE.md`, and acting on any of the fixes above.
