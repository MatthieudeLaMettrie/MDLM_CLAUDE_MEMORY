---
name: project-data-api-governance-courses
description: Five self-study courses written 2026-10-13 — client data APIs + governance for Fabric, Databricks, Snowflake, BigQuery, plus a cloud identity/tokens/secrets security course; where they live
metadata:
  type: project
---

On 2026-10-13 Matthieu asked for full courses (detailed examples + explanations) on exposing warehouse/semantic-model data to clients via APIs and on governance, for Fabric, Databricks, Snowflake and BigQuery, then a security course on managed identities, service principals, service accounts, Entra ID, OAuth, tokens, client IDs/secrets and vaults across all platforms (5 real examples per topic; RBAC/RLS/OLS/least-privilege module added on request).

Saved copies (markdown exports; diagrams only in the live docs):
- [Fabric](courses/fabric-data-apis-and-governance.md) — live: https://claude.ai/code/artifact/d0201f53-da3b-4837-9af2-15bc399d7c73
- [Databricks](courses/databricks-data-apis-and-governance.md) — live: https://claude.ai/code/artifact/9575f10a-3253-4b05-9374-05df8d2e6b97
- [Snowflake](courses/snowflake-data-apis-and-governance.md) — live: https://claude.ai/code/artifact/26c8e947-0a29-4df1-a791-47b58417e15c
- [BigQuery](courses/bigquery-data-apis-and-governance.md) — live: https://claude.ai/code/artifact/3ebc1a91-a77d-4bec-8a87-9f7f10763dca
- [Security: identity, tokens, secrets](../IT-Security/course-cloud-identity-tokens-secrets.md) — live: https://claude.ai/code/artifact/c21fe269-4c42-4ae0-82ac-ba47a9708299

**Why:** study/teaching material; he wants these kept in this repo, not only in claude.ai.
**How to apply:** if a course is edited later, edit the live doc, re-export markdown here and commit. Content was written from model knowledge (no doc lookups); preview-era syntax is flagged in-text — verify before teaching. Open doc comment on the Fabric course: shared managed identity + code filter vs one SP per client with SQL RLS. Related: [[project-semantic-layers-study]].
