---
name: project-datacertprep-todo
description: "Living list of open DataCertPrep work with dates, cost (free vs API credit) and how to do each item — check and update at the start/end of every DataCertPrep session"
metadata:
  node_type: memory
  type: project
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-24T04:02:32.202Z
---

Last updated **2026-09-24**. Tick items off (move to "Done") rather than
deleting, so the thread survives. "Free" = no Anthropic API credit (see
[[feedback-api-spend]] — API-spending work needs Matthieu's explicit go-ahead).

## Dated / time-sensitive

| Due | Item | Cost | How |
|---|---|---|---|
| **2026-09-29** | Confirm MLA-C01 page shows "retired" (last exam 09-28) | free | Load /certs/aws-mla-c01; banner derives from `retirement.lastExamDate` in `lib/cert-data.ts`, no code change expected |
| **After 2026-10-19** | Recheck DP-600/DP-700 blueprints (GitHub issue #46). Live Microsoft page already shows "skills measured as of Oct 19" — change log says minor changes | free | Fetch Microsoft Learn study guides, compare with `content/certs/microsoft-dp-6|700.json`; keep 30–35% range wording |
| **Early 2027** | MLA-C02 standard exam GA (beta from 2026-09-29) — decide whether to add the cert | new content = API credit | Only with explicit budget |

## Question quality

| Added | Item | Cost | How |
|---|---|---|---|
| 09-23 | Review **139 accuracy concerns** in `docs/audits/question-rebalance-concerns.md` ("select TWO" with 3 defensible answers, stale AI-tool facts, wrong explanation details) | free (in-session, verify each against docs) | Fix confirmed ones in `content/questions/**`, then `import-content.ts <cert> --publish` |
| 09-24 | Archive 1 stale duplicate question each in DP-700 and PL-300 | free | `import-content.ts microsoft-dp-700 --publish --prune` (archives, never deletes) — same for PL-300 |
| 09-23 | Length bias: correct answer still longest **~66%** of the time | API credit (or slow manual in-session batches) | `rebalance-questions.ts` default mode; only with a dollar-estimated pilot + cap |
| 09-24 | ~2,150 questions still unshuffled (ambiguous letter wording); small certs still skewed (Tableau 75% "A", AZ-305 61% "A") | cheap API (`--explanations-only`, Haiku) or manual | Needs go-ahead |
| 09-22 | Astronomer Airflow Fundamentals: ≥8 questions use deprecated Airflow-2 `schedule_interval` (exam is Airflow 3) | free (manual) | Edit `content/questions/astronomer-airflow-fundamentals/*`, re-import |
| 09-23 | GH-300: PR-summary/code-review sections sit in the privacy/safeguards guide (`content/guides/github-copilot/prs-projects-admin.mdx`) — only title/weight fixed | free | Move sections to the right domain guide |
| 09-24 | GH-600 (`github-agentic-ai-developer`) last verified 2026-08-10, only 46 single-answer questions | recheck free; more content = API | |

## Curriculum / exam-update alerts

| Added | Item | How |
|---|---|---|
| before 09-22 | Open automated issues **#8, #36, #38, #39, #43** "Exam guide update detected — review required" — never reviewed in these sessions | `gh issue view <n>`; verify against vendor page; close or fix |
| before 09-22 | Issue **#21** Verify Snowflake domain weightings from gated study guides | Needs the gated PDFs |
| 09-22 | Possible Tableau cert rename ("Salesforce Certified Tableau Desktop Foundations") — unconfirmed | Check Salesforce/Tableau page |

## Site, SEO, trust

| Added | Item | Cost |
|---|---|---|
| 09-23 | Public guide previews (even the free guide needs login → Google can't index it) | free |
| 09-23 | Auto-published AI articles go live unreviewed (source of the AI-103 error) — add review gate or slow cadence | free |
| 09-23 | Add named reviewers + primary-source citations to articles | free |
| 07-10 (from repo CLAUDE.md) | Stripe annual plan ($79/yr): confirm `STRIPE_PRICE_ID_ANNUAL` is set in Vercel — status unknown | free |

## Housekeeping

| Added | Item |
|---|---|
| 09-24 | Windows blocks `esbuild.exe` (`spawn EPERM`) → `npx tsx` broken; ask IT / check Windows Security quarantine. Workaround in [[reference-datacertprep-infra]] |
| 09-23 | Repo `CLAUDE.md` status section stale (stops 2026-07-10, says 12 certs / sonnet-4-6) |
| 09-23 | Create global `~/.claude/CLAUDE.md` (repo CLAUDE.md references a non-existent `~/CLAUDE.md`) — Matthieu deferred to a later session |
| 09-23 | 14 pre-existing ESLint errors (e.g. `ExamRunner.tsx` react-hooks/refs, `lib/articles.tsx` unescaped quotes) |
| 09-23 | Generator default model is stale (`claude-sonnet-4-6` in `scripts/generate-content.ts`) |

## Done (keep for the thread)

| Date | Item |
|---|---|
| 09-22 | Study guide / slides / cheatsheet redesign for all certs (PR #44), homepage "Just updated" (PR #45) |
| 09-22 | 3-round curriculum audit; 10 empty domains + 9 missing guides generated |
| 09-22 | Prod outages fixed (unmerged PR, stale Vercel `DATABASE_URL`) |
| 09-23 | aws-mls-c01 report labelled retired |
| 09-23 | Neon DB password rotated + synced to Vercel |
| 09-23 | ASTRA-6 factual + SEO fixes (PR #47) |
| 09-23 | Vercel Speed Insights (PR #48), enabled in dashboard |
| 09-23 | Answer-bias tooling, partial Opus rewrite, free shuffle (PR #49), imported |
| 09-24 | A–D labels + letter-remap shuffle (PR #50), imported, 11,013/11,013 verified |
