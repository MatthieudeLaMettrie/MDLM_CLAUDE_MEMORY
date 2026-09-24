# 2026-09-24: Truth & Testimony, from idea to live site

Matthieu asked for a Christian website and YouTube channel with these parts:
- pointers to apologists (GodLogic, Sam Shamoun, David Wood, Raymond Ibrahim…) and their books;
- positive Bible verses;
- a section on harmful Quran and hadith passages;
- uplifting kids' anime about Jesus.

See [[project-truth-and-testimony]] and [[project-truth-and-testimony-todo]].

**Plan.** Made in plan mode, then approved.
- Choices he made: EN + FR, a separate kids' brand and channel, and an AI-assisted anime pipeline.
- He asked "Firebase or Vercel?" and I recommended Vercel.
- Brand names he picked: "Truth & Testimony" and "Light of the World Kids".
- The DNS check showed truthandtestimony.com and .org are taken.

**Build.**
- Next 16 app with `app/[lang]` routes, a `proxy.ts` locale redirect, hreflang and a sitemap.
- Content lives as typed TS files in `content/`.
- Key design choice: no scripture text is ever written by hand. `scripts/fetch-texts.ts` pulls it from these sources and fails if any reference doesn't resolve:
  - Bible: getbible.net, KJV + Louis Segond 1910;
  - Quran: api.quran.com, Saheeh / Pickthall / Abdel Haleem / Hamidullah;
  - hadith: fawazahmed0 hadith-api on jsDelivr, since sunnah.com is behind Cloudflare.

  A `mustContain` list also checks every claim the prose makes about a source's wording. It caught nothing wrong, but it guards future edits.
- 5 topics: Aisha's age, Q 9:5/9:29, Q 4:34, Q 5:33 + apostasy, abrogation.
- Each topic has these parts:
  - primary text;
  - classical reading;
  - critique;
  - fair Muslim responses with a reply to each.
- Machine gotchas:
  - curl needs `--ssl-no-revoke`, and Node needs `NODE_OPTIONS=--use-system-ca`;
  - the French hadith edition is a community translation from English, and the site says so.

**Design.**
- Used styles.refero.design as inspiration for 3 directions on a Design canvas (https://claude.ai/artifact/1H4KYhw7r2wTfXYWiph2ir):
  - A Vellum (cream editorial);
  - B Vigil (midnight);
  - C Dossier (typeset).
- He liked A but said the cream "feels Claude vibe", so I made Stone / Sage / White / Night backgrounds.
- He picked Stone and asked for images and video. I added public-domain paintings (Bloch's Sermon on the Mount, Rembrandt's Storm and Prodigal Son, Caravaggio's Thomas) and click-to-play YouTube embeds.
- Then he compared Stone, Cream and Midnight and chose **Cream + Midnight**. They were built as light and dark themes: Midnight follows the OS setting, and a header toggle saves the choice in localStorage, applied by an inline head script so there's no flash.

**Video curation finding.** Scraping the channels' latest uploads showed Sam Shamoun's channel attacking David Wood and GodLogic ("This is Why David Wood Must Go", "Exposing GodLogic's Grift").
- So videos are hand-picked in `content/videos.ts`, not an auto feed.
- I also dropped a GodLogic podcast video whose thumbnail showed a cracking mosque dome, since it mocks a sacred symbol.
- The YouTube RSS feeds returned 404 from this machine.

**Ship.**
- Pushed to github.com/MatthieudeLaMettrie/Truth-Testimony (private, `main`).
- Created the Vercel project `matts-projects-6fc9737c/truth-testimony`; the git connection is auto-linked.
- Live at https://truth-testimony.vercel.app.
- Set `NEXT_PUBLIC_SITE_URL` to the vercel.app URL so canonicals don't point at the unbought domain.
- Gotcha: `vercel link` appended `.env*` to `.gitignore`, which re-ignores `.env.example`, so I reverted it.
- The Chrome extension wasn't connected all session, so nothing was checked visually in a real browser. It was verified via build output, curl and the prerendered HTML.
