---
name: project-truth-and-testimony
description: "Truth & Testimony: Matthieu's bilingual Christian apologetics site + YouTube channel + separate kids' anime brand; state as of 2026-09-24"
metadata:
  node_type: memory
  type: project
  originSessionId: 32887d1c-cb9d-43fd-a2f1-1e9ff31b335f
  modified: 2026-09-24T07:29:54.819Z
---

Started 2026-09-24. Plan: `~/.claude/plans/i-want-to-set-tranquil-treehouse.md`.
Repo: `C:\Users\mettrma\Documents\GitHub\truth-and-testimony` → github.com/MatthieudeLaMettrie/Truth-Testimony (private, branch `main`, first pushed 2026-09-24).
- **Live since 2026-09-24:** https://truth-testimony.vercel.app. Vercel project `matts-projects-6fc9737c/truth-testimony` is git-connected, so a push to `main` deploys.
- Env `NEXT_PUBLIC_SITE_URL` is set to the vercel.app URL for now. Change it when the custom domain is bought.
- The Vercel CLI isn't on PATH; use `npx vercel`. `vercel link` appends `.env*` to `.gitignore`, which re-ignores `.env.example`, so revert that.

**Decisions made by Matthieu:**
- EN + FR.
- Apologetics brand "Truth & Testimony / Vérité & Témoignage".
- Kids' anime is a SEPARATE brand + channel, "Light of the World Kids", set to Made for Kids (COPPA). It never cross-links to the Islam-critique content.
- Anime made with an AI-assisted pipeline (Claude scripts + ElevenLabs connector).
- Stack: Vercel (he asked "Firebase or Vercel?" and I recommended Vercel).

**Domains:** truthandtestimony.com and .org are TAKEN. .net, .fr and truth-and-testimony.com looked free (DNS check only). lightoftheworldkids.com/.fr looked free. Nothing is registered yet.

**Built so far:** a Next 16 site covering verses, apologists, 5 Islam topic pages, videos and books. All scripture comes from APIs via `npm run fetch-texts`, which also checks the citations.

**Why:** the Islam section's credibility depends on exact citations, and misquotes get clipped by Muslim debaters.

**How to apply:**
- Never write Quran/hadith/Bible text from memory. Add refs to `content/` and run the fetch script.
- Critique texts, not people (YouTube policy + French loi 1881).
- Any paid AI generation for the anime needs a $ estimate and his go-ahead first, see [[feedback-api-spend]].

**Design (chosen 2026-09-24):**
- Final pick: Cream as the light theme (warm paper, oxblood accent) and Midnight as the dark theme (near-black, candle gold), with Fraunces + Newsreader fonts and an editorial layout. Dark-mode devices get Midnight automatically, and a header toggle lets visitors choose (saved in localStorage). He tried Stone (blue-grey) along the way.
- The site uses public-domain paintings (Bloch, Rembrandt, Caravaggio) in `public/media`, with credits in `content/art.ts`.
- The design canvas is at https://claude.ai/artifact/1H4KYhw7r2wTfXYWiph2ir.

**Videos are hand-picked** in `content/videos.ts`, deliberately not an auto "latest uploads" feed.
- **Why:** as of 2026-09-24, Sam Shamoun's channel posts attacks on David Wood and GodLogic.
- Two items were also rejected: a GodLogic podcast thumbnail showing a cracking mosque dome (mocks a sacred symbol), and the YouTube RSS feed (it returned 404).
- **How to apply:** check each new video's title and thumbnail before adding it.

**Project docs live in the repo** (`docs/PLAN.md`, `docs/DECISIONS.md`, `docs/TODO.md`, plus the `design/` folder of HTML mockups). They are the shareable copy. The dated to-do in memory is [[project-truth-and-testimony-todo]]; keep the two in sync.

**Open items:** see [[project-truth-and-testimony-todo]]. History is in `TruthAndTestimony/sessions/`.
