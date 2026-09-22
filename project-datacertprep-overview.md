---
name: project-datacertprep-overview
description: "What DataCertPrep is, its stack, and major decisions/state as of 2026-09-23"
metadata:
  node_type: memory
  type: project
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-22T23:43:05.258Z
---

DataCertPrep (datacert-prep.com, github.com/MatthieudeLaMettrie/DataCertPrep,
private repo, clones to `C:\Users\mettrma\Documents\GitHub\DataCertPrep`) is a
certification-prep platform covering 47-48 certs across AWS, Azure, GCP,
Microsoft, Databricks, Snowflake, dbt, HashiCorp, GitHub, Confluent,
Astronomer, Tableau, plus 3 in-house AI courses. Next.js 16 (App Router) +
TypeScript strict + Tailwind v4, Prisma 7 + `@prisma/adapter-pg` → Neon
Postgres (prod) / PGlite (local dev), NextAuth v5, Stripe, deployed on Vercel
(project `matts-projects-6fc9737c/datacertprep`), auto-deploys on push to
`master`. Content (questions/flashcards/guides) is generated offline via
`scripts/generate-content.ts` against the Anthropic API and imported to
Postgres — the running app never calls an LLM at request time.

**Why this matters for future work**: as of this session the study
guide/slides/cheatsheet UI was fully redesigned into shared, data-driven
components (`components/guides/GuideExperience.tsx`,
`components/slides/StudyDeck.tsx`, `components/cheatsheet/CheatSheet.tsx`),
replacing a 33-cert hand-coded Gamma-iframe/bespoke-infographic setup. A full
curriculum-currency audit was also done (see
`docs/audits/2026-09-22-curriculum-currency-audit.md` in the repo) — several
certs' domain blueprints were corrected against live vendor pages, and every
domain across every cert now has at least one real question/flashcard/guide.

**Known open items, not yet resolved**:
- `astronomer-airflow-fundamentals` has ≥8 published questions using
  deprecated Airflow-2 syntax (`schedule_interval`) even though the real
  exam now runs on Airflow 3 — flagged in that cert's `guideVersion`, needs
  a question-level rewrite pass, deliberately not done unilaterally.
- GitHub issue #46 tracks rechecking `microsoft-dp-600`/`dp-700` after
  Microsoft's announced 2026-10-19 Fabric blueprint revision.
- `content/reports/aws-mls-c01.md` is kept live but should carry a clear
  retirement notice (added `app/certs/[slug]/report/page.tsx` banner for
  this per his explicit "keep it, just label it clearly" decision).
- An independent Codex/ASTRA-6 content audit on 2026-09-23 found real
  factual errors in guides (AWS DEA-C01, MS DP-700, dbt) and a systemic
  answer-length/position bias in generated questions (correct answer was
  the longest option 69% of the time in a sample) — see
  [[project-datacertprep-content-audit-2026-09-23]] for the full list and
  recommended fix order. Not yet actioned.

**Recurring quirk**: a scheduled GitHub Action auto-publishes an article to
`master` periodically (`content: auto-publish DataCertPrep article
[skip ci]`), which causes ordinary `git push` to `master` to get rejected
with "fetch first" fairly often. Fix is always just
`git pull --rebase origin master` then push again — it's never a real
conflict with anything in `app/`, `lib/`, or `content/certs|questions|
flashcards|guides`.
