---
name: reference-dns-basics
description: "Plain-language glossary of DNS and email-authentication terms (A, CNAME, TXT, MX, SPF, DKIM, TTL, apex domain), written from the FEI domain incident"
metadata:
  node_type: memory
  type: reference
  originSessionId: 5bec0e83-6fe8-42ff-93f7-9ff21a4aeb9c
  modified: 2026-09-24T02:03:54.577Z
---

Written at Matthieu's request while fixing [[2026-09-24-fei-dns-email-fix]].
General knowledge, not project-specific — kept as a standing reference for
future domain/DNS work on any project.

**DNS (Domain Name System)**: the internet's phone book. It translates a
human-readable domain name (`frenchexpatsinvestment.com.au`) into the
technical information other systems need to reach it — an IP address for web
traffic, a mail server for email, a proof-of-ownership string for
verification, and so on. A domain's "DNS records" are entries in that
domain's zone, usually managed through whoever hosts the DNS for it (in the
FEI case, Squarespace, even though the actual website is hosted elsewhere on
Firebase — the registrar/DNS host and the hosting provider don't have to be
the same company).

**Record types actually used in the FEI fix:**

- **A record** — points a hostname straight at an IPv4 address. `A @ →
  199.36.158.100` means "the domain itself (see apex, below) is served at
  this address" — in this case, Firebase Hosting's shared IP.
- **CNAME record** — points a hostname at *another hostname* instead of an
  IP, e.g. `CNAME www → fei-website-f16b0.web.app` means "www.yourdomain
  is really just an alias for this Firebase-provided address." CNAMEs are
  not allowed on the bare apex domain by the DNS standard, which is why
  apex domains need an A record instead.
- **MX record (Mail eXchange)** — tells the world which mail server handles
  incoming email for the domain. It has a **priority** number (lower = tried
  first); a domain should generally have exactly one active mail provider's
  MX record(s). Having two different providers' MX records at once (as FEI
  did — Google's `smtp.google.com` plus a leftover Microsoft-style
  placeholder) confuses mail delivery.
- **TXT record** — a free-text field attached to a hostname, used for
  anything that isn't a pointer to an address. Three unrelated uses showed
  up in this incident alone:
  - **Domain ownership verification** (Firebase's `hosting-site=...` value,
    Google's `google-site-verification=...`) — proves you control the DNS,
    so the platform will let you claim the domain.
  - **SSL certificate verification** (`_acme-challenge` TXT) — proves
    ownership again, specifically so a certificate authority (Let's Encrypt,
    in Firebase's case) will issue an HTTPS certificate for the domain.
  - **Email authentication** (SPF, DKIM — below).
- **SPF (Sender Policy Framework)** — a TXT record at the domain's apex
  (`v=spf1 include:_spf.google.com ~all`) listing which mail servers are
  allowed to send email *claiming to be from* this domain. Receiving mail
  servers check it to reject spoofed/forged email. **A domain can only have
  one SPF record** — if you need to authorize multiple senders, they go
  inside one record, not multiple records.
- **DKIM (DomainKeys Identified Mail)** — a TXT record (at a special
  hostname like `google._domainkey.yourdomain.com`) holding a public
  cryptographic key. The mail provider signs every outgoing message with the
  matching private key; recipients use the public key in this record to
  verify the message wasn't altered in transit and really came from an
  authorized sender.
- **TTL (Time To Live)** — how long (in seconds/minutes/hours) other servers
  are allowed to cache a DNS record before re-checking it. This is *why DNS
  changes aren't instant* — a record with a 4-hour TTL can take up to 4
  hours to be seen everywhere after you change it, even though the change
  saved immediately on your end.

**Apex / root domain**: the bare domain with no subdomain prefix —
`frenchexpatsinvestment.com.au` itself, as opposed to `www.
frenchexpatsinvestment.com.au`. Apex domains have DNS restrictions (no
CNAME, as above) that often make them slightly more fiddly to point at a
hosting provider than a `www` subdomain is — which is why many setups
(including FEI's) just redirect the apex to `www` once both are configured.

**Domain verification / ownership proof**: the general pattern where a
platform (Firebase, Google Workspace, etc.) asks you to add a specific,
unique TXT record before it will let you claim a domain or issue it a
certificate — proving you control the domain's DNS, not just that you typed
its name somewhere. This is exactly what went wrong in the FEI incident: the
verification TXT record must live in the DNS zone of the *exact* domain
being verified, not a similarly-named one.
