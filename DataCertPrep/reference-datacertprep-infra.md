---
name: reference-datacertprep-infra
description: "Where DataCertPrep's infrastructure lives and machine-specific gotchas for working with it"
metadata:
  node_type: memory
  type: reference
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-24T04:02:55.857Z
---

- **Repo**: github.com/MatthieudeLaMettrie/DataCertPrep (private). Local
  clone: `C:\Users\mettrma\Documents\GitHub\DataCertPrep`.
- **Production**: www.datacert-prep.com, Vercel project
  `matts-projects-6fc9737c/datacertprep`. Auto-deploys on push to `master`.
- **Database**: Neon Postgres. `.env.vercel-prod` in the repo root (gitignored,
  local only) holds the live prod `DATABASE_URL` — he explicitly said to
  leave this file in place for reuse across sessions. Prisma CLI scripts need
  `DATABASE_ADAPTER=neon-ws` to route over WebSockets (port 443) rather than
  raw TCP 5432, which is unreliable/blocked from some networks — see
  `scripts/db-adapter.ts` and the repo's own `docs/RUNBOOK.md`.
- **Content-generation model**: historically `GENERATION_MODEL=claude-opus-5`
  ("never Haiku for questions"). **Since 2026-09-23 no API-spending run
  without his explicit go-ahead + dollar estimate — see [[feedback-api-spend]].**
  `claude-opus-5-5` rejects forced `tool_choice` (use `auto`). Key is
  `ANTHROPIC_API_KEY` in `.env` (not `.env.local`).
- **Publishing content to prod** (free, DB only):
  `set -a; source .env.vercel-prod; set +a; DATABASE_ADAPTER=neon-ws <run> scripts/import-content.ts <cert> --publish`
  — upserts by id = sha1(cert, domainId, stem), so editing options/explanations
  keeps ids; `--prune` archives (never deletes) stale rows. The auto-mode
  classifier blocks this until he Shift+Tabs out of auto mode.

**Machine-specific gotchas (this Windows box)**:
- `gh` and `vercel` CLIs are installed but not on PATH in fresh Bash tool
  calls — prefix with `export PATH="$PATH:/c/Program Files/GitHub CLI"` or
  use `npx vercel`.
- Plain `curl` to external HTTPS sites fails with a schannel revocation-check
  error on this network — add `--ssl-no-revoke`.
- `pdftoppm`/poppler-utils isn't installed, so the Read tool's PDF `pages`
  param (page-range rendering) fails on large PDFs — omit `pages` and read
  the whole file instead; it still works via the doc-embedding path.
- **Since ~2026-09-23 17:00 Windows blocks `node_modules\@esbuild\win32-x64\esbuild.exe`**
  (`spawn EPERM` / "Permission denied") — likely antivirus/AppLocker. Then
  `npx tsx …` exits 0 **with no output** (silent failure!). Don't bypass the
  block. Workaround: Node 24 native type-stripping plus a resolve hook that
  adds `.ts` to extensionless relative imports. Save as any `.mjs` file:
  ```js
  import { registerHooks } from "node:module";
  registerHooks({ resolve(s, c, next) {
    try { return next(s, c); } catch (e) {
      if (s.startsWith(".") && !/\.[cm]?[jt]s$/.test(s)) return next(`${s}.ts`, c);
      throw e; } } });
  ```
  Run: `node --no-warnings --import "file:///<path-to-hook>.mjs" scripts/<x>.ts …`
  (vitest: `node … node_modules/vitest/vitest.mjs run`). Scratch scripts must
  live inside the repo (e.g. gitignored `.rebalance-batches/`) to resolve packages.
- Stopping a background `xargs -P` job does **not** kill its node children on
  Windows. Verify/kill via PowerShell:
  `Get-CimInstance Win32_Process | ? { $_.CommandLine -match '<script>' }`.
- WebFetch has its own ~15-minute cache; re-fetching a URL you already hit
  earlier in the same session can silently return stale content — use plain
  `curl` (with `--ssl-no-revoke`) for a guaranteed-fresh check instead.
