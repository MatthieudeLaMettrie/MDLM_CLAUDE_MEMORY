---
name: feedback-verify-with-real-execution
description: "Before claiming a fix works, actually run it (real CLI/API call, real streamed endpoint) — he tests personally and catches gaps with concrete evidence"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 0859047a-8d76-42cf-9934-2604488a2604
  modified: 2026-09-28T07:01:57.293Z
---

Verify fixes by actually executing the real thing, not just static checks (typecheck/lint/
build/unit tests passing). Confirmed repeatedly during the 2026-09-28 Agentic OS session
([[project-agentic-os]]): he uses the app himself in a real browser and reports back concrete,
specific evidence — a screenshot of an actual error overlay, an exact quoted response the agent
gave, a precise UI element circled. Every one of those reports pointed at something static
checks alone would have missed (a permission flag that's semantically valid but functionally a
no-op, a hydration mismatch invisible to `next build`, a CLI behavior difference from what the
docs/`--help` implied).

**Why:** static checks confirm the code *compiles and follows the types*, not that it *does
the right thing when actually run*. This session had several bugs that passed tsc/eslint/vitest/
`next build` cleanly and still didn't work: Claude Code's `--permission-mode default` silently
denying tool calls, Copilot's `-s`/`--no-ask-user` flags doing nothing for tool permission,
`--allowedTools` needing exact tool names discovered by testing (not guessed from docs),
Copilot's `web_fetch` needing a *separate* `--allow-all-urls` on top of the tool allow. None of
that surfaces without actually spawning the real CLI and reading real output.

**How to apply:** before reporting a fix as done, run the real command/endpoint it changes —
not a mock, not just the type signature. For CLI-wrapping code specifically: test the exact CLI
invocation directly first (confirms the flag/mechanism actually works), then again through the
app's own endpoint end-to-end (confirms the wiring). If the fix can't be executed in this
session (e.g. no browser automation available), say so explicitly rather than implying it was
visually confirmed — he asked to "check" the animation fix specifically because I couldn't
demonstrate it, and later independently found a real hydration bug in that same area.
