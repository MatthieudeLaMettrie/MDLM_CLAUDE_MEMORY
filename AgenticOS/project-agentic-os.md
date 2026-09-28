---
name: project-agentic-os
description: "Agentic OS — Matthieu's personal hub centralising all AI subscriptions/APIs (prompt box, model picker, usage/limits, spend guard); v1 on next-app (PR #1); HUD dashboard + conversation continuation on feat/hud-dashboard (PR #2, 2026-09-28)"
metadata:
  node_type: memory
  type: project
  originSessionId: 2c89c9aa-932f-4d51-9330-8806b4b62ed0
  modified: 2026-09-28T07:01:44.639Z
---

Personal web app: one prompt box + model picker + usage/limits for Claude Code, Claude API,
Codex, Copilot, Gemini (app+API), ElevenLabs, Replicate, OpenArt.

**Repo**: github.com/MatthieudeLaMettrie/MDLM_Agentic_OS (private), clone at
`C:\Users\mettrma\Documents\GitHub\MDLM_Agentic_OS`. Docs in `docs/` (PLAN, RESEARCH,
DECISIONS, RUNBOOK) are the source of truth — read them first.
State 2026-09-25: v1 built on branch `next-app`, **draft PR #1** awaiting his review/merge.
His original Vite UI mockup lives in `prototype/`; its look was ported (he chose that).
`Documents\GitHub\agentic-os` is a leftover scratch scaffold (superseded; ask before deleting).

Architecture: one VPS he owns (not created yet) running Next.js 16 + node:sqlite in Docker
behind Caddy; subscription CLIs run headless there; API calls go through the spend guard.

**Why:** the PC he uses Claude Code on (C:\Users\mettrma, Win 11 Enterprise) is a
**work laptop** — no personal API keys/subscription logins may live on it; secrets only in the VPS `.env`.
**How to apply:** local dev uses the mock provider only (ENABLE_MOCK_PROVIDER=1). Real-provider
smoke tests need explicit go-ahead per [[feedback-api-spend]]. Next steps: he creates the VPS +
DNS, then follow docs/RUNBOOK.md; verify Docker build + each provider on the server.

**2026-09-28**: he designed a second UI mockup as a claude.ai artifact — a dark sci-fi "HUD"
look (animated particle engine core per provider, gauge widgets, vault map overlay, terminal
panels) — and asked to have it built into the repo. Ported it as `src/components/Hud.tsx` +
`src/components/hud/` (EngineCore canvas, VaultOverlay) on branch `feat/hud-dashboard`, **draft
PR #2** (base `next-app`, since PR #1 is still open there). It's now the app's default landing
view; the classic tabbed UI is untouched, reachable via a "Classic view" link. All decorative
numbers in the artifact (fake DataCertPrep/T&T KPIs, canned terminal transcripts, auto-routing
copy) were replaced with real data from this app's own `/api/usage`, `/api/history`,
`/api/templates` — confirmed with him via AskUserQuestion before building (new-home-dashboard +
real-metrics, both recommended options). Verified with tsc/eslint/vitest/next build and curl
smoke tests of the streaming `/api/run` contract; **no visual browser check** — the Claude in
Chrome extension wasn't connected in that session, so he should eyeball `npm run dev` before
merging PR #2.

**Same day, continued — real bugs he caught by actually using it** (PR #2 grew to 8 commits):
- Engine core canvas was permanently static on his machine: `prefers-reduced-motion` was
  re-checked on every pause/resume click, so once the OS/browser has that on, "Resume motion"
  could never win. Fixed by deciding it once on mount instead.
- That same mount-time `matchMedia`/`new Date()` read inside `useState` initializers caused a
  **hydration mismatch** (server has no `window`, renders at a different instant) — a distinct
  bug from the one above, surfaced by a pasted React error overlay. Fixed with the standard
  SSR-safe pattern: fixed placeholder value + set the real one in a post-mount effect.
- Clock overlapped the "Classic view" link (both absolutely-positioned in the same corner).
- Engine switcher had Claude Code/Codex/**Copilot**; his real three are Claude Code/Codex/
  **Gemini** — swapped it, and added a "not configured" indicator since this laptop has no
  `GEMINI_API_KEY` (by design, [[project-agentic-os]] laptop policy above).
- **"Chat only" mode couldn't call any tool at all**, including read-only ones like web fetch —
  `--permission-mode default` (Claude) / non-interactive Copilot both deny anything needing
  approval with no TTY to answer it, so the agent would just say "I need permission" and stop.
  Fixed by pre-approving a read-only allow-list per CLI (Claude: `--allowedTools
  WebFetch,WebSearch,Read,Grep,Glob`; Copilot: `--allow-tool=view,grep,glob,web_fetch,web_search
  --allow-all-urls`, plus discovered Copilot's `-s`/`--no-ask-user` flags it already had do
  nothing for tool permission — only output formatting). Codex's `read-only` sandbox already
  allowed web tools, no fix needed there.
- **No conversation continuity** — every Run was a stateless one-shot CLI call, so an agent's
  clarifying question could never actually be answered. Added real session continuation for
  all three engines, each via its own native mechanism: Claude Code `--resume <id>` (id read
  from the CLI's own `result.session_id`), Codex `exec resume <id>` (id read from its
  `thread.started` event), Copilot `--session-id=<uuid>` (Copilot never prints an id anywhere,
  so the app assigns its own `crypto.randomUUID()` up front — same flag both creates and
  resumes). New `runs.session_id` DB column + `RunEvent` "session" type + `resumeSessionId` on
  run requests, all provider-agnostic; UI (Prompt Router banner, History's "Continue
  conversation" button, HUD ask box) needed zero provider-specific code as a result.

Every fix above was verified by actually running the real CLI (`claude`/`codex`/`copilot`
installed on this machine) directly, then again through the live `/api/run` endpoint — not
just tsc/eslint/build passing. See [[feedback-verify-with-real-execution]].
