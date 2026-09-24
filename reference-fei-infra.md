---
name: reference-fei-infra
description: "French Expats Investment (FEI) website — Firebase project, domains, DNS host, and email provider"
metadata:
  node_type: memory
  type: reference
  originSessionId: 5bec0e83-6fe8-42ff-93f7-9ff21a4aeb9c
  modified: 2026-09-24T04:00:39.086Z
---

**Project**: French Expats Investment (FEI) — website at
`www.frenchexpatsinvestment.com.au` (primary public URL) and
`www.frenchexpatsinvestment.com`. Distinct from [[project-datacertprep-overview]]
— unrelated business.

**Hosting**: Firebase Hosting, project `fei-website-f16b0`
(console.firebase.google.com/project/fei-website-f16b0). Default URLs are
`fei-website-f16b0.web.app` / `fei-website-f16b0.firebaseapp.com`. Two custom
domains connected: `www.frenchexpatsinvestment.com` and
`www.frenchexpatsinvestment.com.au`, each with the bare apex domain set to
redirect to its `www` subdomain.

**DNS + registrar**: Both `frenchexpatsinvestment.com` and
`frenchexpatsinvestment.com.au` are registered *and* DNS-hosted at
Squarespace (`account.squarespace.com/domains/managed/<domain>/dns/dns-settings`).
They are two entirely separate domains/zones despite the near-identical name
— **easy to edit the wrong one**, since Squarespace's domain-switcher dropdown
truncates long names identically for both. Always check the browser URL or
the exact dropdown value before editing records. See
[[2026-09-24-fei-dns-email-fix]] for the incident this caused.

**Email**: Google Workspace (Gmail), managed at
`admin.google.com/ac/domains/manage`, not Microsoft 365 — despite a stray
leftover `ms18838980.msv1.invalid` MX record (Squarespace's inert default
placeholder) that had to be deleted from both domains' DNS to stop it
conflicting with Google's MX record.

**Correct DNS record set for each domain's zone** (apex `@`):
- `A @ → 199.36.158.100` (Firebase Hosting apex IP)
- `CNAME www → fei-website-f16b0.web.app`
- `MX @ → 1 smtp.google.com` (single record, priority 1 — no second MX)
- `TXT @ → v=spf1 include:_spf.google.com ~all` (SPF, one only per domain)
- `TXT google._domainkey → v=DKIM1; k=rsa; p=...` (DKIM, generated per-domain
  in Google Admin under Authenticate email)
- `TXT @ → hosting-site=fei-website-f16b0` (Firebase apex ownership
  verification — only needed if the apex domain isn't yet Connected)
- `TXT _acme-challenge → ...` (Firebase SSL cert verification — same
  condition)

As of 2026-09-24: both domains have this full correct MX/SPF/DKIM set
(Google Admin shows "All ok" email setup status for both). Firebase apex
verification for `frenchexpatsinvestment.com.au` was still pending re-check
at last update — see [[2026-09-24-fei-dns-email-fix]] for current status.
