# 2026-09-24 — FEI apex domain + email DNS troubleshooting

Matthieu shared screenshots of a Firebase Hosting + Squarespace DNS problem
for `frenchexpatsinvestment.com.au` (French Expats Investment site — see
[[reference-fei-infra]]). Diagnosed from screenshots only; no browser access
used this session.

**Problem 1 — apex domain stuck on "Needs setup" in Firebase.**
`www.frenchexpatsinvestment.com.au` was already Connected, but the bare apex
`frenchexpatsinvestment.com.au` (which Firebase redirects to `www`) wouldn't
verify. Root cause: Matthieu has two near-identical, separately-registered
domains in Squarespace — `frenchexpatsinvestment.com` and
`frenchexpatsinvestment.com.au` — and Squarespace's domain-switcher dropdown
truncates both to the same visible text. The Firebase verification TXT
record (`hosting-site=fei-website-f16b0`) and the SSL `_acme-challenge` TXT
record got typed into the **`.com` zone** with the **full `.com.au` hostname
pasted into the Name field**, e.g. Name =
`_acme-challenge.frenchexpatsinvestment.com.au` while editing the `.com`
zone — which Squarespace then appends to *that* zone's own apex, producing a
meaningless record that verified nothing.

Fix: switch to the actual `.com.au` zone in the dropdown (confirmed by
reading the record's Name field in the edit panel, not just the truncated
dropdown label), re-add both TXT records there with short Names (`@` and
`_acme-challenge`), and remove three stale Squarespace parking `A` records
(199.36.158.101/.102/.103) that Firebase's setup wizard flagged for removal.
Also added the real Firebase apex `A` record (`199.36.158.100`) matching the
pattern already working for the `.com` domain. Confirmed via screenshot that
`.com.au` now has the correct record set. Firebase re-verification was
triggered but not yet confirmed as "Connected" by end of session — **open
item**.

Leftover mis-scoped TXT records on the `.com` zone (harmless, since that
domain was already verified) were also cleaned up on request — Name changed
from the full pasted domain to `@`.

**Problem 2 — broken email deliverability on both domains.**
Google Workspace Admin (`admin.google.com/ac/domains/manage`) flagged
"Action needed" for both `frenchexpatsinvestment.com` (primary) and
`frenchexpatsinvestment.com.au` (secondary). Cause: each domain had **two
competing MX records** — the correct `smtp.google.com` (at the wrong
priority, 10 instead of 1) plus a leftover inert
`ms18838980.msv1.invalid` record (a Squarespace default placeholder, not an
actual Microsoft 365 setup) — plus **no SPF** and **no DKIM** at all.

Fix applied to both `frenchexpatsinvestment.com` and
`frenchexpatsinvestment.com.au` (same steps on each zone in Squarespace):
- Deleted the `ms18838980.msv1.invalid` MX record.
- Changed the remaining MX priority from 10 to 1.
- Generated a DKIM key in Google Admin (Gmail → Authenticate email) and
  added the resulting `google._domainkey` TXT record.
- Added the SPF TXT record: `v=spf1 include:_spf.google.com ~all` at `@`
  (checked first that no duplicate SPF TXT existed — a domain can only have
  one).

**Resolved**: Google Admin's Manage domains page now shows "All ok" for
email setup status on both `frenchexpatsinvestment.com` (primary) and
`frenchexpatsinvestment.com.au` (secondary) — confirmed 2026-09-24.

**Open items for next session:**
1. Confirm `frenchexpatsinvestment.com.au` apex now shows "Connected" in
   Firebase Hosting (not just "Needs setup") — this was the original issue
   that started the session; not yet re-checked since the DNS fix.

Also see [[reference-dns-basics]] — a plain-language DNS/A/CNAME/MX/TXT/
SPF/DKIM glossary written this session at Matthieu's request, using this
exact incident as the worked example.
