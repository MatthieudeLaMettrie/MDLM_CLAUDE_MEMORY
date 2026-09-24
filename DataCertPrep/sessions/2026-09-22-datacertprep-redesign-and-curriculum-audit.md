# 2026-09-22/23 — Study experience redesign, curriculum audit, two prod incidents

Marathon session on DataCertPrep (datacert-prep.com). Rough chronology:

1. **Study experience redesign.** Matthieu shared a Claude Design canvas
   mockup for a new study-guide/slides/cheatsheet UI and asked for it applied
   across all 47-48 certs, not just the one in the mockup. Built shared,
   data-driven components (`GuideExperience`, `StudyDeck`, `CheatSheet`)
   reading from `content/certs/<slug>.json` via a new `lib/cert-content.ts`,
   replacing a 33-cert hand-coded Gamma-iframe/bespoke-infographic setup
   (`app/certs/[slug]/infographic/page.tsx` went from ~5420 lines to ~40).
   Iterated once on the slides deck after feedback that it looked sparse
   ("nothing but the titles"). Shipped as PR #44 (later PR #45 for a
   follow-up).

2. **Curriculum currency audit.** Asked to check whether all certs reflect
   current, accurate exam blueprints. Ran a 3-round audit (see
   `docs/audits/2026-09-22-curriculum-currency-audit.md` in the repo),
   fixing confirmed drift (AWS DVA-C02, 3 GitHub certs' exam codes/domains,
   Azure AB-100 level, dbt/Snowflake domain restructuring) and generating
   content for domains that had none — 241 questions / 100 flashcards / 10
   guides, plus 9 more missing guides in a later pass. Standing rule I held
   throughout: only edit live data on primary-source evidence or ≥2
   independent sources converging exactly — logged as
   [[feedback-autonomy-and-verification-bar]]. Left 2 items genuinely
   unconfirmed (Tableau possible rename; GitHub issue #46 tracks a
   2026-10-19 Fabric/DP-600/DP-700 recheck).

3. **Two production incidents, neither caused by this session's shipped
   code.** First: prod was still serving the pre-redesign UI because PR #44
   had never actually been merged — a misleading local `ETIMEDOUT` repro
   briefly pointed at a DB networking issue before the real cause (unmerged
   PR) was found. Second, right after merging: prod-wide 500s traced to a
   Vercel `DATABASE_URL` secret that was 102 days stale; fixed via
   `vercel env rm/add` + redeploy. Recurred a second time the same session
   when Matthieu rotated the actual Neon password locally without telling
   Vercel — same fix applied again.

4. **Homepage promotion.** Mid-session ask to show off the redesign and
   promote new content on the homepage for SEO — added a "Just updated"
   section with ItemList JSON-LD to `app/page.tsx`.

5. **Retired-cert report decision.** `aws-mls-c01`'s exam is retired but its
   paid report was still live; asked Matthieu via AskUserQuestion whether to
   pull it — he said keep it but label it clearly, so a retirement banner
   was added to `app/certs/[slug]/report/page.tsx`.

6. **Memory infrastructure.** Closed the session by setting up this very
   repo — turning Claude Code's native, already-auto-loaded memory folder
   into a git repo and pushing it to a new private GitHub repo
   (`MDLM_CLAUDE_MEMORY`), mirroring the value of his existing Codex memory
   repo (`MDLM_CODEX_MEMORY`) — see [[reference-claude-memory-setup]] for
   how the two differ.

**Notable non-technical moment**: Matthieu pasted the live prod Neon DB
password into chat directly (twice, across this and a prior session) despite
a suggested safer file-write pattern. Handled without echoing it back;
flagged plainly; he rotated the password later in the session. Full detail
in [[feedback-autonomy-and-verification-bar]].

---

**Continued in** [2026-09-23 → 24 session](2026-09-23-datacertprep-accuracy-fixes-and-answer-bias.md) (ASTRA-6 fixes, answer bias, cost incident). Open work: [[project-datacertprep-todo]].
