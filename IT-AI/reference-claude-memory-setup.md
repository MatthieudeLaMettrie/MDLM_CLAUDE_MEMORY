---
name: reference-claude-memory-setup
description: "How Claude Code's persistent memory works on this machine and how this GitHub-backed repo layers on top of it"
metadata:
  node_type: memory
  type: reference
  originSessionId: d2649431-2b1c-4569-b6af-6a7ec31f61b4
  modified: 2026-09-24T05:52:27.921Z
---

**Native memory (Claude Code's own feature, not something I built)**: Claude
Code auto-loads a per-user memory folder at
`C:\Users\mettrma\.claude\projects\C--Users-mettrma\memory\` at the start of
every session on this machine. `MEMORY.md` in that folder is always loaded
in full; it's a short index of one-line pointers to the other `.md` files,
which are loaded on demand when relevant. Each memory file has YAML
frontmatter (`name`, `description`, `metadata.type` ∈ user/feedback/project/
reference) and can cross-link other memory files with `[[slug]]` syntax. This
mechanism existed before this session — what didn't exist was any git/GitHub
layer on top of it.

**What I added on 2026-09-23**: turned that same folder into a git repo
(`git init` in place, not a separate directory) and pushed it to a new
**private** GitHub repo, `MDLM_CLAUDE_MEMORY`
(github.com/MatthieudeLaMettrie/MDLM_CLAUDE_MEMORY). This means:
- The memory Claude Code actually reads every session **is** the git
  working tree — no sync step, no separate copy to keep in mind.
- Changes get committed and pushed like any other repo, giving Matthieu a
  version history and an off-machine backup of what Claude knows about him
  and his projects.
- A `sessions/` subfolder (one per topic, e.g. `FEI/sessions/`,
  `DataCertPrep/sessions/`) holds dated journal entries, for narrative
  history that doesn't belong in the always-loaded topic files.

**Topic folders (added 2026-09-24)**: topic files live under per-topic
directories — `FEI/`, `DataCertPrep/`, `IT-Website/`, `IT-AI/` — with
genuinely cross-project files (`user-profile.md`,
`feedback-autonomy-and-verification-bar.md`, `feedback-api-spend.md`)
staying at the repo root alongside `MEMORY.md`. `MEMORY.md`'s links are the
only thing that need to reflect a file's path; internal `[[slug]]`
cross-links between memory files are resolved by name, not path, so they
didn't need updating when files moved. One caveat I noticed but haven't
fully confirmed: root-level memory files pick up auto-maintained frontmatter
fields (`metadata.node_type`, `originSessionId`, `modified`) that files
under `sessions/` never get — possibly evidence of some indexing step keyed
to directory depth or location. If a topic file stops getting its
frontmatter refreshed after being moved into a topic folder, that's the
likely cause; worth a closer look if memory recall seems to degrade for a
moved file.

**How this differs from his Codex setup** (`MDLM_CODEX_MEMORY`, also
private): that repo uses `AGENTS.md` as the instructions file Codex reads
explicitly each session (Codex has no built-in auto-loading memory
mechanism, so the repo *is* the whole mechanism — he has to point Codex at
it). This Claude repo has no `AGENTS.md` equivalent because Claude Code
already auto-loads `MEMORY.md` and the topic files natively; the GitHub layer
here is purely for backup/versioning/visibility, not for making memory work
in the first place. Both repos share the `MEMORY.md` index concept and a
`sessions/` dated-journal convention.

**Raw session transcripts are a separate, unrelated thing**: Claude Code
also writes full `.jsonl` transcripts of every session to
`C:\Users\mettrma\.claude\projects\<project-path-hash>\<session-uuid>.jsonl`.
Those are large, not human-readable, not summarized, and not part of this
memory repo — they're Claude Code's own raw history mechanism, kept
per-project rather than per-user, and not backed up anywhere.
