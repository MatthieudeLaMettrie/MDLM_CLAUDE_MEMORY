---
name: project-truth-and-testimony-todo
description: "Living list of open Truth & Testimony work with owner, cost and how-to. Check and update at the start and end of every Truth & Testimony session."
metadata:
  node_type: memory
  type: project
  originSessionId: 32887d1c-cb9d-43fd-a2f1-1e9ff31b335f
  modified: 2026-09-24T07:30:07.022Z
---

Last updated **2026-09-24**. Tick items off (move them to "Done") rather than deleting them, so the history survives.
- "Free" means no paid API or generation credit. Paid work needs Matthieu's explicit go-ahead and a $ estimate, see [[feedback-api-spend]].
- The repo copy is `docs/TODO.md` in github.com/MatthieudeLaMettrie/Truth-Testimony. Keep both in sync.

## Needs Matthieu (accounts, money, personal data)

| Item | Cost | How |
|---|---|---|
| Buy the domain: `truthandtestimony.net` or `truth-and-testimony.com` (the .com and .org are taken). Optionally also `lightoftheworldkids.com` | ~$10–15/yr each | Any registrar. The 2026-09-24 check was DNS only, so confirm at the registrar |
| Apply to Amazon Associates, US + FR (amazon.fr "Partenaires") | free | Needs a live site with content, which it now has. Give the tags to Claude |
| Create the YouTube channels: "Truth & Testimony" and "Light of the World Kids" (set Made for Kids) | free | Needs his Google account. Keep the two channels separate, never cross-link |
| Ask the apologists for clip permission and cross-promotion | free | Email or DM. Protects against copyright strikes on the commentary channel |

## Claude can do (on request)

| Item | Cost | How |
|---|---|---|
| Connect the custom domain once it's bought | free | `npx vercel domains add <domain>`, set the DNS records at his registrar, then update env `NEXT_PUBLIC_SITE_URL` and redeploy |
| Add the Amazon tags | free | Vercel env `NEXT_PUBLIC_AMAZON_TAG_US` / `_FR`, then redeploy |
| Submit the sitemap to Google Search Console | free | Needs his Google login to verify. DNS TXT verification works once the domain is live |
| Grow the verse library to about 100 per language | free | Add refs to `content/sources.ts`, then `npm run fetch-texts` |
| More Islam topics (e.g. Q 8:12, Muhammad and the Satanic verses, Quran preservation) | free | Same template and editorial rules as `content/islam-topics.ts`. Hand-check every citation |
| Curate more videos | free | Check each title and thumbnail; see the comment in `content/videos.ts` |
| Kids' anime pilot script (Nativity), EN + FR | free | Claude writes it |
| Kids' anime pilot production | **paid** (ElevenLabs credits) | Give a $ estimate and wait for a go-ahead before any generation |
| Replace the monogram circles with apologist photos | free, but needs rights | Only with permission from each apologist; their photos are copyrighted |

## Done
- 2026-09-24: plan approved. Next.js site built (EN/FR, 39 pages) with a citation-check script.
- 2026-09-24: design explored. 3 directions, then background options, then Cream + Midnight chosen, with paintings and videos.
- 2026-09-24: pushed to github.com/MatthieudeLaMettrie/Truth-Testimony and live at https://truth-testimony.vercel.app, with auto-deploy on push to `main`.
