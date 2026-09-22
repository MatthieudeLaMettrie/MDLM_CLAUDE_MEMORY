---
name: user-profile
description: "Who the user is, their role, and how they work with Claude Code"
metadata:
  node_type: memory
  type: user
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-22T23:16:57.220Z
---

Matthieu (email mdlm@hotmail.fr) is the solo founder/operator of DataCertPrep
(datacert-prep.com), a live SaaS certification-prep platform (Next.js 16 +
Prisma + Neon Postgres, deployed on Vercel). He does full-stack work himself —
product, engineering, content pipeline, deployment — and treats Claude Code as
a hands-on engineering collaborator he directs through multi-hour, multi-step
sessions rather than a one-shot code generator.

He works on Windows (PowerShell primary, Git Bash tool also used) and is
comfortable with me installing tooling on his machine directly (winget for
`gh`/`vercel` CLIs) when it's needed to finish a task.

**How to work with him**: he moves fast and gives very short commands ("go",
"go ahead", "proceed", "do 1 and 4") expecting me to carry full context
forward without re-explaining. He's fine with me executing multi-step,
production-affecting work (DB writes, deploys, content generation, git
merges) once he's given a clear go-ahead — but see [[feedback-autonomy-and-verification-bar]]
for where he still wants a pause. He reads detailed structured reports
(tables, headers) closely and acts on specific line items from them (e.g.
"do 1 and 4" referring to items in a numbered list I'd just given him).

He already runs a similar persistent-memory setup for OpenAI Codex sessions
— a private GitHub repo (`MDLM_CODEX_MEMORY`) with `AGENTS.md` (instructions),
`MEMORY.md` (distilled memory), and `sessions/` (dated session-summary
journal). He asked for the Claude Code equivalent on 2026-09-23 — see
[[reference-claude-memory-setup]] for what was built.
