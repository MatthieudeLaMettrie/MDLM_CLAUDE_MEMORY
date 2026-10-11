---
name: project-semantic-layers-study
description: Semantic layer / metrics / AI agent study material for Databricks + Snowflake in MDLM_CODEX_MEMORY; two paste-and-run labs still awaiting the user's run
metadata:
  type: project
---

Between 2026-10-09 and 2026-10-12 I wrote, in `MDLM_CODEX_MEMORY/guides/data-engineering/` (local clone `C:\Users\mettrma\Documents\GitHub\MDLM_CODEX_MEMORY`, follow its AGENTS.md: session note + MEMORY.md + commit/push every session):
- `SEMANTIC_LAYERS_DATABRICKS_SNOWFLAKE_GUIDE.md`: one fixture as a Databricks metric view and a Snowflake semantic view.
- `SEMANTIC_LAYER_AI_AGENTS_AND_PUBLISHING_GUIDE.md`: Genie Agent and Cortex Agent, plus publishing to Teams, SharePoint, a website, Power BI and Tableau.
- `SEMANTIC_LAYER_VS_SEMANTIC_MODEL_SCENARIOS.md`: nine scenarios.
- `semantic-layers-lab/databricks_metric_view_lab.sql` (23 checks) and `snowflake_semantic_view_lab.sql` (21 checks + deliberate grain failure).

Expected lab answers: net 880.00, 4 orders, AOV 220.00, East 680.00.

**Why:** the user is studying for SnowPro GenAI and Databricks and wants study material verified against real execution ([[feedback-verify-with-real-execution]]).

**How to apply:** when the user pastes PASS/FAIL grids, compare them with the expected answers, fix the guides and record the results in a CODEX session note. This work laptop has no Databricks CLI, no Snowflake connection and no Chrome extension. The user chose paste-and-run to keep personal trial credentials off it ([[project-agentic-os]] notes this is a work PC). Still unverified: the Databricks `format` block, whether metric-view consumers need SELECT on the source tables, the Snowflake role-playing pattern, where the warehouse goes in `CREATE AGENT`, and the Genie API field names.
