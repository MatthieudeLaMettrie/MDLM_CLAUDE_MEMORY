# Databricks — Data APIs & Governance: Full Course

> Source of truth: the live Claude Doc (https://claude.ai/code/artifact/9575f10a-3253-4b05-9374-05df8d2e6b97), which also holds the drawn diagrams. Exported 2026-10-13. Written from model knowledge; preview-era syntax is flagged in the text for verification.


Oct 13, 2026 · @Matt

## Course overview

By the end of this course you can serve Unity Catalog data and governed metrics to external clients through REST APIs, Databricks Apps or Delta Sharing, and govern the platform with Unity Catalog, system tables and account-level controls.

**Audience:** data engineers, platform engineers and analytics leads who know SQL and Python and have used a Databricks workspace.

**Prerequisites:** a Unity Catalog–enabled workspace (Premium tier or above on Azure, AWS or GCP), account-admin access (or a friendly account admin), a serverless or Pro SQL warehouse, and the Databricks CLI v0.2xx+.

**Running scenario.** *ContosoData* sells retail sales analytics. Each client must pull only its own data — raw rows and governed KPIs — into its own systems. The data lives in Unity Catalog under `contoso_prod.gold`.

### Learning outcomes

1. Explain account, workspace, metastore and catalog, and where each control is enforced.
2. Choose between the Statement Execution API, Databricks Apps, Delta Sharing, a custom API and Lakebase for a client need.
3. Build each option with OAuth machine-to-machine authentication.
4. Isolate clients with row filters, column masks and ABAC policies.
5. Govern with Unity Catalog privileges, tags, lineage, system tables and budgets.
6. Ship changes with Asset Bundles and CI/CD behind network controls.

### Module map

| # | Module | Type | Time |
| --- | --- | --- | --- |
| 1 | Architecture essentials | Concepts | 45 min |
| 2 | API options compared | Concepts | 30 min |
| 3 | Statement Execution API | Lab | 60 min |
| 4 | FastAPI in Databricks Apps | Lab | 90 min |
| 5 | Delta Sharing to clients | Lab | 60 min |
| 6 | External FastAPI + metric views | Lab | 2 h |
| 7 | Securing the API | Concepts + lab | 60 min |
| 8 | Unity Catalog foundations | Concepts + lab | 60 min |
| 9 | Fine-grained access | Lab | 90 min |
| 10 | Audit, monitoring, cost | Lab | 60 min |
| 11 | ALM, identity, network | Concepts + lab | 60 min |
| 12 | Capstone + quiz | Project | 3 h |

### Shared setup script

Run this once in a SQL editor attached to a SQL warehouse. Every lab reuses it.

```sql
CREATE CATALOG IF NOT EXISTS contoso_prod;
CREATE SCHEMA IF NOT EXISTS contoso_prod.gold;
CREATE SCHEMA IF NOT EXISTS contoso_prod.api;
CREATE SCHEMA IF NOT EXISTS contoso_prod.sec;

CREATE OR REPLACE TABLE contoso_prod.gold.sales (
  sale_id BIGINT, client_id INT, sale_date DATE, store STRING,
  product STRING, qty INT, amount DECIMAL(18,2), customer_email STRING);

INSERT INTO contoso_prod.gold.sales VALUES
 (1,1,'2026-09-01','Paris-01','Coffee',3,12.60,'a@x.com'),
 (2,1,'2026-09-02','Paris-02','Tea',1,4.20,'b@x.com'),
 (3,2,'2026-09-01','Lyon-01','Coffee',5,21.00,'c@y.com');

CREATE OR REPLACE VIEW contoso_prod.api.v_sales AS
SELECT sale_id, client_id, sale_date, store, product, qty, amount
FROM contoso_prod.gold.sales;
```

Databricks ships features quickly, and several named here (Apps, Lakebase, ABAC, metric views, Genie API) were in preview at various points. Check their status in your cloud and region before you teach them.

## Module 1 — Databricks architecture essentials

Databricks splits governance across two planes. The **account** owns identities, metastores, network and billing. The **workspace** owns compute, notebooks, jobs and apps. **Unity Catalog** sits in between and governs every table, view, volume, model and function for all workspaces attached to it.

### 1.1 The hierarchy

| Level | What it is | Admin | Key controls |
| --- | --- | --- | --- |
| **Account** | Your Databricks tenancy in one cloud | Account admin | Users/groups/SPs (SCIM), metastores, network configs, budgets, Delta Sharing recipients |
| **Metastore** | Top Unity Catalog container, one per region | Metastore admin (better: a group) | Catalog creation rights, storage root, sharing |
| **Workspace** | Deployment where people work | Workspace admin | Compute, SQL warehouses, Apps, Git folders, IP access lists, entitlements |
| **Catalog** | First namespace level, usually per environment or domain | Catalog owner | Grants, workspace binding, managed storage location |
| **Schema** | Second level | Schema owner | Grants, default storage |
| **Securable** | Table, view, metric view, volume, function, model | Object owner | Grants, row filter, column mask, tags, comments |

Objects are addressed with a **three-level namespace**: `catalog.schema.object`, for example `contoso_prod.gold.sales`.

### 1.2 Compute that serves data

| Compute | Serves | Use for APIs? |
| --- | --- | --- |
| **SQL warehouse** (serverless, Pro, Classic) | SQL over JDBC/ODBC, the Statement Execution API, dashboards | Yes, the main query engine for APIs. Prefer **serverless** for fast start and auto-stop |
| All-purpose / jobs clusters | Notebooks, Spark jobs | No (slow start, not request/response) |
| **Databricks Apps** | Python/Node web apps (FastAPI, Flask, Streamlit, Dash, Gradio) hosted on serverless compute | Yes, for APIs that live inside Databricks |
| **Model Serving** | ML models, feature and function endpoints | Yes, for real-time predictions or features |
| **Lakebase** (managed Postgres) | Low-latency OLTP reads and writes, synced from Delta tables | Yes, for millisecond lookups at high concurrency |

### 1.3 Identities and authentication

| Identity | Use | Auth method |
| --- | --- | --- |
| User | People, interactive tools | SSO (OIDC/SAML) + MFA from your IdP |
| **Service principal** | APIs, jobs, CI/CD, client integrations | **OAuth M2M** (client ID + secret), or Entra ID SP on Azure |
| Group | Grants and entitlements | Synced from your IdP with SCIM |
| Federated workload | External system (GitHub Actions, a client's IdP) | **OAuth token federation**: exchange the external JWT for a Databricks token, with no Databricks secret |

OAuth machine-to-machine token request (workspace level):

```bash
curl -s -u "$CLIENT_ID:$CLIENT_SECRET" \
  https://<workspace-host>/oidc/v1/token \
  -d "grant_type=client_credentials&scope=all-apis"
# → {"access_token":"eyJ...","token_type":"Bearer","expires_in":3600}
```

**Personal access tokens (PATs)** still work, but they're tied to a user and are long-lived. Restrict or disable them for production (Module 11) and use OAuth instead.

### 1.4 Where data physically lives

- **Managed tables** are stored in the catalog/schema managed location (your cloud storage) and fully lifecycle-managed by Unity Catalog. Prefer them.
- **External tables** point to an **external location** (path + storage credential). Use them when other tools must write the files.
- **Volumes** govern non-tabular files.

### 1.5 Exercise

1. Run the shared setup script.
2. In **Catalog Explorer**, open `contoso_prod.gold.sales` and look at the **Permissions**, **Lineage** and **Details** tabs.
3. In account console → **User management → Service principals**, create `sp-client-northwind`, generate an **OAuth secret**, and call the token endpoint above.

## Module 2 — API options compared

Databricks has more native ways to serve data than Fabric. **Delta Sharing** is the standout: an open protocol built for giving external organisations governed, live access to tables without building an API at all.

### 2.1 Side-by-side comparison

| Criterion | Statement Execution API | Databricks Apps (FastAPI) | Delta Sharing | External FastAPI + gateway | Lakebase behind an API | JDBC/ODBC direct |
| --- | --- | --- | --- | --- | --- | --- |
| Hosted in | Databricks (REST) | Databricks serverless | Databricks (share server) | Your cloud | Databricks Postgres + your API | n/a |
| Protocol | REST, SQL in JSON | REST you design | Delta Sharing REST (open) | REST you design | Postgres wire + REST | JDBC/ODBC/ADBC |
| Result shape | JSON / Arrow, inline or external links | Your JSON | Parquet/Delta files, read by pandas, Spark, Power BI, Tableau… | Your JSON | Your JSON | Rows |
| Client auth | Databricks OAuth / token | Databricks OAuth | Bearer token file or OIDC federation, or Databricks-to-Databricks | API keys, OAuth — your choice | Your choice | Databricks OAuth |
| Quotas, keys, portal | No | Build it yourself | Per-recipient access, no quotas | Yes (APIM / AWS API Gateway / Apigee) | Yes, through the gateway | No |
| Latency | Seconds | Seconds | Batch / files | Seconds | Milliseconds | Seconds |
| Best for | Internal apps, simple integrations | Custom endpoints without extra cloud infra | Clients wanting the whole dataset | Commercial client APIs | High-QPS lookups | Technical clients, BI tools |

### 2.2 What each option is

**SQL Statement Execution API** (`/api/2.0/sql/statements`): POST a SQL string and parameters, get rows back. It runs on a SQL warehouse and enforces Unity Catalog permissions of the caller. Think of it as "JDBC over HTTPS".

**Databricks Apps** host a Python or Node web app on serverless compute. Each app gets its own **service principal**, can declare **resources** (a SQL warehouse, secrets, serving endpoints, Lakebase) and can run queries either as the app identity or **on behalf of the calling user**.

**Delta Sharing** shares tables, views, volumes and models with **recipients**:

- *Databricks-to-Databricks*: the client's Unity Catalog mounts your share as a catalog, with no tokens.
- *Open sharing*: any client (pandas, Spark, Power BI, Tableau, Excel, Snowflake readers) gets a credential file or uses **OIDC federation** with their own IdP.

You can revoke a recipient instantly, and every access is audited.

**External FastAPI + API gateway** uses `databricks-sql-connector` with a service principal. It's the same pattern as in the Fabric course, needed when you sell an API product.

**Lakebase** is managed Postgres inside Databricks. **Synced tables** copy Delta tables into Postgres for millisecond reads, so an app or API can serve high-concurrency lookups.

**Metric views** (a Unity Catalog semantic layer) define measures and dimensions in YAML once. Any of the options above can query them with `MEASURE()` so that clients get the same KPI definitions as your dashboards.

**Genie API** lets an application send natural-language questions to a curated Genie space and get SQL + results back. It's useful for conversational client portals, not for deterministic integrations.

### 2.3 Decision tree

&#91;embedded content: Databricks API option decision tree · 4 questions, 5 outcomes\]

Stop at the first "yes". Whichever option you land on, query metric views when the client needs KPIs, so every route returns the same definitions.

### 2.4 Common mistakes

- **Using a PAT of a human** in a client integration. When that person leaves, the integration breaks, and the audit log shows them as the actor.
- **Pointing APIs at an all-purpose cluster.** Use a serverless SQL warehouse: it starts in seconds, scales and stops itself.
- **Sharing base tables through Delta Sharing** without a row filter or a per-client view. Share per-client views, or use dynamic views in the share.
- **Returning huge results inline.** Use `disposition: EXTERNAL_LINKS` for large result sets in the Statement Execution API.

## Module 3 — Lab: SQL Statement Execution API

The Statement Execution API lets any HTTP client run parameterised SQL on a SQL warehouse, as a service principal, with Unity Catalog permissions enforced. In this lab the client `sp-client-northwind` pulls its sales over HTTPS.

### 3.1 Grant the minimum

```sql
-- Unity Catalog privileges (run as catalog owner)
GRANT USE CATALOG ON CATALOG contoso_prod TO `sp-client-northwind`;
GRANT USE SCHEMA  ON SCHEMA  contoso_prod.api TO `sp-client-northwind`;
GRANT SELECT      ON VIEW    contoso_prod.api.v_sales TO `sp-client-northwind`;
```

Then, in the SQL warehouse's **Permissions**, give the service principal **Can use**. In the workspace, also make sure it has the *Databricks SQL access* entitlement.

Service principals are referenced in SQL by their **application ID** (a UUID). Display names work in some UIs; in scripts, use the application ID in backticks.

### 3.2 Request anatomy

```http
POST https://<workspace-host>/api/2.0/sql/statements
Authorization: Bearer <oauth-token>
Content-Type: application/json

{
  "warehouse_id": "abcdef1234567890",
  "statement": "SELECT sale_id, sale_date, store, product, qty, amount FROM contoso_prod.api.v_sales WHERE sale_date >= :from_date ORDER BY sale_date",
  "parameters": [ { "name": "from_date", "value": "2026-09-01", "type": "DATE" } ],
  "wait_timeout": "30s",
  "on_wait_timeout": "CONTINUE",
  "disposition": "INLINE",
  "format": "JSON_ARRAY",
  "row_limit": 10000
}
```

| Field | Meaning |
| --- | --- |
| `parameters` | Named parameters bound server-side (`:from_date`). Never concatenate values into the SQL string |
| `wait_timeout` | 5–50 s. If the query finishes in time you get results synchronously |
| `on_wait_timeout` | `CONTINUE` returns a `statement_id` to poll; `CANCEL` aborts |
| `disposition` | `INLINE` (up to about 25 MiB, JSON) or `EXTERNAL_LINKS` (presigned URLs to result chunks, up to about 100 GiB) |
| `format` | `JSON_ARRAY`, `ARROW_STREAM` or `CSV` (the last two only with external links) |

### 3.3 Python client (raw HTTP)

```python
import time, requests

HOST = "https://adb-1234567890.12.azuredatabricks.net"
WAREHOUSE = "abcdef1234567890"

def token(client_id, secret):
    r = requests.post(f"{HOST}/oidc/v1/token", auth=(client_id, secret),
                      data={"grant_type": "client_credentials", "scope": "all-apis"})
    r.raise_for_status()
    return r.json()["access_token"]

def run(sql, params, tok):
    H = {"Authorization": f"Bearer {tok}"}
    r = requests.post(f"{HOST}/api/2.0/sql/statements", headers=H, json={
        "warehouse_id": WAREHOUSE, "statement": sql, "parameters": params,
        "wait_timeout": "30s", "on_wait_timeout": "CONTINUE",
        "disposition": "INLINE", "format": "JSON_ARRAY"}).json()
    sid = r["statement_id"]
    while r["status"]["state"] in ("PENDING", "RUNNING"):
        time.sleep(2)
        r = requests.get(f"{HOST}/api/2.0/sql/statements/{sid}", headers=H).json()
    if r["status"]["state"] != "SUCCEEDED":
        raise RuntimeError(r["status"])
    cols = [c["name"] for c in r["manifest"]["schema"]["columns"]]
    rows = r["result"].get("data_array", [])
    nxt = r["result"].get("next_chunk_internal_link")
    while nxt:                                    # multi-chunk inline results
        c = requests.get(f"{HOST}{nxt}", headers=H).json()
        rows += c.get("data_array", [])
        nxt = c.get("next_chunk_internal_link")
    return [dict(zip(cols, row)) for row in rows]

tok = token("<sp app id>", "<oauth secret>")
print(run("SELECT * FROM contoso_prod.api.v_sales WHERE sale_date >= :d",
          [{"name": "d", "value": "2026-09-01", "type": "DATE"}], tok))
```

Note that `JSON_ARRAY` returns every value as a **string** (or null). Cast types on the client using `manifest.schema.columns[].type_name`.

### 3.4 Same thing with the Databricks SDK

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.sql import StatementParameterListItem

w = WorkspaceClient(host=HOST, client_id="<sp app id>", client_secret="<secret>")  # OAuth M2M handled for you
resp = w.statement_execution.execute_statement(
    warehouse_id=WAREHOUSE,
    statement="SELECT * FROM contoso_prod.api.v_sales WHERE sale_date >= :d",
    parameters=[StatementParameterListItem(name="d", value="2026-09-01", type="DATE")],
    wait_timeout="30s")
print(resp.result.data_array)
```

### 3.5 Large results

With `"disposition": "EXTERNAL_LINKS", "format": "ARROW_STREAM"`, the response contains `external_links[]` with presigned cloud-storage URLs that expire after a short time. Download them **without** the Databricks `Authorization` header (they're presigned), and read them with `pyarrow.ipc.open_stream`.

### 3.6 Lab check

- [ ] The service principal gets rows; a second service principal without `SELECT` gets `PERMISSION_DENIED`.
- [ ] Passing `'; DROP TABLE ...` as a parameter value returns zero rows, not an error from the injection.
- [ ] The query shows in **Query history** under the service principal's name.
- [ ] The warehouse auto-stops after its idle timeout.

## Module 4 — Lab: FastAPI hosted in Databricks Apps

Databricks Apps run your FastAPI code on serverless compute next to your data, with no container registry, load balancer or VM to manage. Authentication is Databricks OAuth, so callers must be identities in your account. That makes it ideal for partners you onboard as service principals or users, and less ideal for anonymous key-based public APIs.

### 4.1 Two authorisation models

| Model | Query runs as | Unity Catalog enforces | Use when |
| --- | --- | --- | --- |
| **App authorization** | The app's own service principal (auto-created) | The app SP's grants | All callers see the same data, or your code filters by caller |
| **User authorization** (on-behalf-of) | The calling user or SP, through the forwarded token | The *caller's* grants, row filters and masks | Per-caller isolation enforced by Unity Catalog |

With user authorization, the app receives the caller's token in the `X-Forwarded-Access-Token` header. You choose which scopes (for example `sql`) the app may use on the user's behalf.

### 4.2 Project files

```
client-api/
├── app.yaml
├── requirements.txt
└── app.py
```

`app.yaml`:

```yaml
command: ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
env:
  - name: WAREHOUSE_ID
    valueFrom: sql-warehouse        # resource key declared on the app
```

`requirements.txt`:

```
fastapi
uvicorn
databricks-sql-connector
databricks-sdk
```

`app.py`:

```python
import os
from datetime import date
from fastapi import FastAPI, Header, HTTPException, Query
from databricks import sql
from databricks.sdk.core import Config

cfg = Config()                                   # picks up the app SP's host / client id / secret from env
HTTP_PATH = f"/sql/1.0/warehouses/{os.environ['WAREHOUSE_ID']}"
app = FastAPI(title="ContosoData Client API (Databricks Apps)")

def connect_as_user(user_token: str):
    """On-behalf-of: Unity Catalog row filters see the CALLER."""
    return sql.connect(server_hostname=cfg.host.replace("https://", ""),
                       http_path=HTTP_PATH, access_token=user_token)

@app.get("/health")
def health():
    return {"ok": True}

@app.get("/v1/sales")
def sales(date_from: date, date_to: date, limit: int = Query(1000, le=10000),
          x_forwarded_access_token: str | None = Header(None)):
    if not x_forwarded_access_token:
        raise HTTPException(401, "User authorization not enabled or no token")
    if (date_to - date_from).days > 366:
        raise HTTPException(400, "Max range 366 days")
    with connect_as_user(x_forwarded_access_token) as cn, cn.cursor() as cur:
        cur.execute("""
            SELECT sale_id, sale_date, store, product, qty, amount
            FROM contoso_prod.api.v_sales
            WHERE sale_date BETWEEN :f AND :t
            ORDER BY sale_date LIMIT :n""",
            {"f": date_from, "t": date_to, "n": limit})
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]

@app.get("/v1/kpis/monthly")
def kpis(year: int, x_forwarded_access_token: str = Header(...)):
    with connect_as_user(x_forwarded_access_token) as cn, cn.cursor() as cur:
        cur.execute("""
            SELECT month, MEASURE(total_sales) AS sales, MEASURE(total_units) AS units
            FROM contoso_prod.api.mv_sales
            WHERE year = :y GROUP BY month ORDER BY month""", {"y": year})
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]
```

The code doesn't filter by client: the **row filter** on the table (Module 9) does it, because the query runs as the caller. `mv_sales` is a metric view, built in Module 6.

### 4.3 Create and deploy

```bash
databricks apps create client-api --description "ContosoData client API"
# UI: App → Edit → Resources → add SQL warehouse, key 'sql-warehouse', permission 'Can use'
# UI: App → Edit → User authorization → add scope 'sql'

databricks sync ./client-api /Workspace/Shared/apps/client-api
databricks apps deploy client-api --source-code-path /Workspace/Shared/apps/client-api
databricks apps get client-api        # shows the URL and the app's service principal
```

Then grant callers access to the app: **App → Permissions →** `sp-client-northwind` **Can use**.

### 4.4 Call it from the client

```python
import requests
tok = requests.post("https://<workspace-host>/oidc/v1/token", auth=("<sp app id>", "<secret>"),
                    data={"grant_type": "client_credentials", "scope": "all-apis"}).json()["access_token"]
r = requests.get("https://client-api-1234567890.aws.databricksapps.com/v1/sales",
                 params={"date_from": "2026-09-01", "date_to": "2026-09-30"},
                 headers={"Authorization": f"Bearer {tok}"})
print(r.status_code, r.json()[:3])
```

The app URL's domain differs by cloud. Copy it from `databricks apps get`. Programmatic calls with OAuth tokens are supported, but check token scope requirements for your release.

### 4.5 Operations

- **Logs:** the app's **Logs** tab streams stdout/stderr. Use structured JSON logging.
- **Scaling and cost:** apps are billed while running, by compute size. Stop dev apps when unused.
- **Secrets:** add a *Secret* resource and read it as an env var with `valueFrom`. Never commit secrets.
- **Network:** apps follow workspace networking. Serverless egress can be restricted with **network policies** (Module 11).

### 4.6 Lab check

- [ ] `/health` returns `ok` via the app URL with an OAuth token, and 401/403 without one.
- [ ] Two service principals with different row-filter mappings get different rows from `/v1/sales`.
- [ ] Query history shows the **caller** (not the app SP) as the query user.

## Module 5 — Lab: Delta Sharing to external clients

When a client wants *the dataset* rather than an endpoint, Delta Sharing beats any API. There's no server for you to run, no pagination for them to code, the data is always live, and you can revoke access in one command. One dynamic view can serve every client, each seeing only its own rows.

### 5.1 Concepts

| Object | What it is |
| --- | --- |
| **Share** | A named, read-only collection of tables, views, volumes, models or notebooks |
| **Recipient** | An external organisation. It's either *Databricks-to-Databricks* (identified by the client's metastore ID) or *open* (a bearer-token credential file, or OIDC federation) |
| **Recipient properties** | Key-value pairs on the recipient (for example `client_id = 1`) that shared views can read with `CURRENT_RECIPIENT()` |
| **Provider** | You, as seen from the recipient's side |

### 5.2 One dynamic view for all clients

```sql
CREATE SCHEMA IF NOT EXISTS contoso_prod.share;

CREATE OR REPLACE VIEW contoso_prod.share.v_client_sales AS
SELECT sale_id, sale_date, store, product, qty, amount
FROM contoso_prod.gold.sales
WHERE client_id = CAST(CURRENT_RECIPIENT('client_id') AS INT);
```

`CURRENT_RECIPIENT('client_id')` returns the property of whoever is reading the share, so the same view yields different rows per recipient.

### 5.3 Create the share and recipients

```sql
CREATE SHARE IF NOT EXISTS client_sales COMMENT 'Per-client sales, daily refresh';
ALTER SHARE client_sales ADD VIEW contoso_prod.share.v_client_sales;

-- Open sharing recipient (client does NOT use Databricks)
CREATE RECIPIENT IF NOT EXISTS northwind
  COMMENT 'Northwind Ltd – contract C-2026-014'
  PROPERTIES ('client_id' = '1');

-- Databricks-to-Databricks recipient (client HAS Unity Catalog)
CREATE RECIPIENT IF NOT EXISTS fabrikam
  USING ID 'azure:westeurope:<client-metastore-uuid>'
  PROPERTIES ('client_id' = '2');

GRANT SELECT ON SHARE client_sales TO RECIPIENT northwind;
GRANT SELECT ON SHARE client_sales TO RECIPIENT fabrikam;

DESCRIBE RECIPIENT northwind;   -- shows the one-time activation link for open sharing
```

Send the **activation link** through a secure channel. The client downloads a `config.share` credential file **once**; the link then stops working.

Options worth knowing:

- `ALTER SHARE ... ADD TABLE t WITH HISTORY` lets recipients use time travel and Change Data Feed for incremental loads.
- `ALTER SHARE ... ADD TABLE t PARTITION (client_id = 1)` shares only matching partitions of a partitioned table.
- Recipient **IP access lists** limit where an open recipient can connect from.
- **Token lifetime** is a metastore setting. Rotate tokens with `ALTER RECIPIENT northwind ROTATE TOKEN`.
- **OIDC federation** lets open recipients authenticate with their own IdP instead of a long-lived bearer token. Prefer it where available.

### 5.4 What the client does

Python (pandas), no Databricks needed:

```python
# pip install delta-sharing
import delta_sharing
profile = "config.share"
print(delta_sharing.SharingClient(profile).list_all_tables())
df = delta_sharing.load_as_pandas(f"{profile}#client_sales.share.v_client_sales")
print(df.head())
```

Spark:

```python
df = spark.read.format("deltaSharing").load("config.share#client_sales.share.v_client_sales")
```

Other readers: the Power BI and Excel Delta Sharing connectors, Tableau, and any tool that speaks the open protocol.

A Databricks client (Fabrikam) runs this once:

```sql
CREATE CATALOG contoso_data USING SHARE contoso_provider.client_sales;
SELECT * FROM contoso_data.share.v_client_sales;
```

### 5.5 Govern and monitor

```sql
-- Who read what, last 7 days
SELECT event_time, request_params.recipient_name AS recipient, action_name,
       request_params.share AS share, response.status_code
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name LIKE 'deltaSharing%'
  AND event_date >= current_date() - 7
ORDER BY event_time DESC;

-- Revoke instantly
REVOKE SELECT ON SHARE client_sales FROM RECIPIENT northwind;
```

The exact `request_params` keys vary by action, so inspect a few raw rows first.

### 5.6 Lab check

- [ ] Northwind's pandas read returns only `client_id = 1` rows (the column isn't even shared).
- [ ] Changing the recipient property to `2` changes the rows they see on the next read.
- [ ] A revoked recipient gets 403 on the next request.
- [ ] Audit rows show each query with the recipient name.

## Module 6 — Lab: external FastAPI + API gateway, with metric views

Use this pattern when you sell an API product: clients get API keys, quotas and a developer portal from a gateway (Azure API Management, AWS API Gateway or Google Apigee), and your FastAPI service queries Databricks as a single service principal. **Metric views** guarantee that the KPIs clients receive match your internal dashboards.

### 6.1 Build the semantic layer: a metric view

```sql
CREATE OR REPLACE VIEW contoso_prod.api.mv_sales
WITH METRICS
LANGUAGE YAML
AS $$
version: 0.1
source: contoso_prod.gold.sales
dimensions:
  - name: client_id
    expr: client_id
  - name: year
    expr: YEAR(sale_date)
  - name: month
    expr: DATE_TRUNC('MONTH', sale_date)
  - name: product
    expr: product
measures:
  - name: total_sales
    expr: SUM(amount)
  - name: total_units
    expr: SUM(qty)
  - name: avg_basket
    expr: SUM(amount) / COUNT(DISTINCT sale_id)
$$;

-- Query it: measures need MEASURE(), aggregation is resolved for any GROUP BY
SELECT month, MEASURE(total_sales) AS sales, MEASURE(avg_basket) AS avg_basket
FROM contoso_prod.api.mv_sales
WHERE client_id = 1 AND year = 2026
GROUP BY month ORDER BY month;
```

Why this matters: `avg_basket` is a ratio. If clients pulled pre-aggregated monthly rows and averaged them, they'd get the wrong annual number. A metric view recomputes the ratio correctly at whatever grain is requested. The YAML spec version and available fields (joins, filters, window measures) evolve, so check the current spec.

### 6.2 Service code

```python
# app/db.py
import os
from databricks import sql
from databricks.sdk.core import Config, oauth_service_principal

HOST = os.environ["DATABRICKS_HOST"]            # adb-xxx.azuredatabricks.net (no https://)
HTTP_PATH = os.environ["DATABRICKS_HTTP_PATH"]  # /sql/1.0/warehouses/<id>

def _creds():
    cfg = Config(host=f"https://{HOST}",
                 client_id=os.environ["DATABRICKS_CLIENT_ID"],
                 client_secret=os.environ["DATABRICKS_CLIENT_SECRET"])
    return oauth_service_principal(cfg)

def query(stmt: str, params: dict) -> list[dict]:
    with sql.connect(server_hostname=HOST, http_path=HTTP_PATH,
                     credentials_provider=_creds) as cn, cn.cursor() as cur:
        cur.execute(stmt, params)                 # named :params are bound server-side
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]
```

```python
# app/main.py
import hmac, os
from datetime import date
from fastapi import FastAPI, Depends, Header, HTTPException, Query
from .db import query

app = FastAPI(title="ContosoData Client API", version="1.0.0")
GATEWAY_SECRET = os.environ["GATEWAY_SECRET"]

def current_client(x_gateway_secret: str = Header(...), x_client_key_id: str = Header(...)) -> int:
    if not hmac.compare_digest(x_gateway_secret, GATEWAY_SECRET):
        raise HTTPException(401)
    rows = query("SELECT client_id FROM contoso_prod.sec.api_subscription WHERE key_id = :k",
                 {"k": x_client_key_id})
    if not rows:
        raise HTTPException(403)
    return rows[0]["client_id"]

@app.get("/v1/sales")
def sales(date_from: date, date_to: date, limit: int = Query(1000, le=10000), offset: int = 0,
          client_id: int = Depends(current_client)):
    return query("""SELECT sale_id, sale_date, store, product, qty, amount
                    FROM contoso_prod.api.v_sales
                    WHERE client_id = :c AND sale_date BETWEEN :f AND :t
                    ORDER BY sale_date, sale_id LIMIT :n OFFSET :o""",
                 {"c": client_id, "f": date_from, "t": date_to, "n": limit, "o": offset})

@app.get("/v1/kpis/monthly")
def kpis(year: int, client_id: int = Depends(current_client)):
    return query("""SELECT month, MEASURE(total_sales) AS sales, MEASURE(total_units) AS units,
                           MEASURE(avg_basket) AS avg_basket
                    FROM contoso_prod.api.mv_sales
                    WHERE client_id = :c AND year = :y
                    GROUP BY month ORDER BY month""", {"c": client_id, "y": year})
```

In production, cache the key-to-client lookup and reuse connections, because opening a Databricks SQL session per request adds latency.

### 6.3 Gateway configuration

| Gateway | Keys | Rate limit / quota | Forward client identity |
| --- | --- | --- | --- |
| Azure API Management | Subscriptions per product | `rate-limit-by-key`, `quota-by-key` | `set-header X-Client-Key-Id = @(context.Subscription.Id)` |
| AWS API Gateway (REST) | API keys + usage plans | Usage plan throttle + quota | Mapping template: `$context.identity.apiKeyId` into a header |
| Google Apigee | Developer apps + API products | Quota, SpikeArrest policies | `AssignMessage` with `client_id` flow variable |

In every case, also inject `X-Gateway-Secret` from the gateway's secret store, and restrict the backend's ingress to the gateway.

### 6.4 Deploy

Package it like any FastAPI service (Container Apps, AWS ECS/App Runner or Cloud Run). Store `DATABRICKS_CLIENT_SECRET` in the cloud's secret manager, or better use **OAuth token federation** so the container's cloud identity is exchanged for a Databricks token and no Databricks secret exists at all.

### 6.5 Lab check

- [ ] `/v1/kpis/monthly` returns the same numbers as an AI/BI dashboard built on `mv_sales`.
- [ ] Two keys return disjoint data.
- [ ] Rate limit triggers HTTP 429.
- [ ] Query history shows one service principal, and gateway logs show which client called.

## Module 7 — Securing the API

The same five rules apply on every platform: authenticate every caller, identify each one individually, isolate rows per caller, keep secrets out of code, and cap what any one caller can consume. This module maps them to Databricks controls.

### 7.1 Least privilege per option

| Option | Identity | Minimum grants |
| --- | --- | --- |
| Statement Execution API | One SP per client | `USE CATALOG`, `USE SCHEMA`, `SELECT` on `api` views; **Can use** on one warehouse; *Databricks SQL access* entitlement |
| Databricks Apps (on-behalf-of) | App SP + caller | App SP: **Can use** on the warehouse resource. Caller: **Can use** on the app + UC grants |
| Delta Sharing | Recipient | `SELECT ON SHARE` only; no workspace access at all |
| External FastAPI | One API SP | `SELECT` on `api` views and `sec.api_subscription`; **Can use** on one warehouse |

Never make an API identity a **workspace admin**, a **metastore admin** or the **owner** of the catalog. Owners implicitly have all privileges and can change grants.

### 7.2 Isolation strategy by option

| Option | Enforced by | Notes |
| --- | --- | --- |
| SP per client (Statement API, Apps OBO) | Unity Catalog **row filter** keyed on `current_user()` or group membership | Strongest. The database refuses other rows even if code is wrong |
| Delta Sharing | `CURRENT_RECIPIENT()` in shared dynamic views | Strong, per recipient |
| One shared SP (external FastAPI) | Code filter on `client_id` + automated cross-client test | Weakest. Mitigate with tests and code review |

Row filter for service-principal-per-client (full syntax in Module 9):

```sql
CREATE TABLE contoso_prod.sec.principal_client (principal STRING, client_id INT);
INSERT INTO contoso_prod.sec.principal_client VALUES
  ('<northwind-sp-application-id>', 1), ('<fabrikam-sp-application-id>', 2);

CREATE OR REPLACE FUNCTION contoso_prod.sec.client_rows(cid INT)
RETURNS BOOLEAN
RETURN is_account_group_member('data-engineers')
    OR EXISTS (SELECT 1 FROM contoso_prod.sec.principal_client p
               WHERE p.principal = current_user() AND p.client_id = cid);

ALTER TABLE contoso_prod.gold.sales SET ROW FILTER contoso_prod.sec.client_rows ON (client_id);
```

For a service principal, `current_user()` returns its **application ID**. Verify this once by running `SELECT current_user()` as the SP.

### 7.3 Secrets and tokens

- **OAuth M2M** with short-lived (1 h) tokens; no PATs for integrations.
- **OAuth secrets** expire (you set the lifetime, maximum about 2 years). Track expiry and rotate by creating a second secret before deleting the first.
- **Token federation** removes Databricks secrets entirely for workloads that have their own IdP (GitHub Actions, Azure managed identity, AWS IAM through OIDC, a client's Entra tenant).
- **Databricks secret scopes** (or Key Vault-backed scopes on Azure) for secrets that jobs and apps need. Read them with `dbutils.secrets.get` or app *Secret* resources.

### 7.4 Cost and abuse controls

- **Dedicated serverless SQL warehouse** for the API (for example "Small", min 1 / max 4 clusters), so client load can't slow internal BI. Set an **auto-stop** time.
- **Query timeouts:** set `STATEMENT_TIMEOUT` on the API warehouse (for example 120 s).
- **Gateway quotas**, and hard caps on rows and date range in code.
- **Tag the warehouse** (`cost_center = client-api`) so `system.billing.usage` attributes its cost.
- **Budgets** with email alerts at account level (Module 10).
- **Query result cache:** identical queries within the cache window return instantly without warehouse compute.

### 7.5 Threat checklist

| Threat | Control |
| --- | --- |
| Client reads another client's rows | Row filter / `CURRENT_RECIPIENT()` / tested code filter |
| SQL injection | Named parameters (`:p`) in every API; never f-strings |
| Stolen client credential | Short OAuth lifetimes, recipient token rotation, IP access lists, gateway key regeneration |
| Exfiltration through notebooks | Restrict who has workspace access; serverless egress network policies |
| Over-privileged API identity | Quarterly review of grants with `system.information_schema.table_privileges` |
| Runaway cost | Dedicated warehouse, max clusters, timeouts, budgets |

### 7.6 Exercise

Write a pytest that obtains tokens for two client SPs, calls every endpoint, and asserts the two result sets share no `sale_id`. Run it in CI after each deployment.

## Module 8 — Unity Catalog governance foundations

Unity Catalog is the single place where Databricks governs data and AI assets: one permission model, one audit log, automatic lineage, and tagging for discovery and policy. Get the catalog layout and ownership right first; everything else builds on them.

### 8.1 Designing the catalog layout

| Pattern | Catalogs | Pros | Cons |
| --- | --- | --- | --- |
| **By environment** | `contoso_dev`, `contoso_test`, `contoso_prod` | Simple isolation; bind each to its workspace | Domains share a catalog |
| **By domain** | `sales`, `finance`, `hr` | Clear ownership | Environments must be separated by schema or workspace binding |
| **Domain × env (recommended at scale)** | `sales_prod`, `sales_dev`, `finance_prod`… | Ownership *and* isolation | More objects to manage. Automate with Terraform or Asset Bundles |

Within a catalog, use schemas per layer or purpose: `bronze`, `silver`, `gold`, `api` (client contract), `share` (Delta Sharing views), `sec` (security tables and functions).

### 8.2 Ownership

- Every securable has exactly one **owner** (a user, SP or group). The owner has all privileges on the object and can grant them.
- **Make groups the owners**, never individuals: `ALTER CATALOG contoso_prod OWNER TO \`grp-sales-data-owners\`;\`
- **Metastore admin** should be a small group, ideally used only through break-glass procedures.

### 8.3 The privilege model

Privileges **inherit downwards**: `SELECT` granted on a catalog applies to every current and future table in it.

| Privilege | On | Allows |
| --- | --- | --- |
| `BROWSE` | Catalog | See metadata in Catalog Explorer without data access (good for discovery and request-access) |
| `USE CATALOG` / `USE SCHEMA` | Catalog / schema | Prerequisite to touch anything inside |
| `SELECT` | Table, view, metric view, schema, catalog | Read data |
| `MODIFY` | Table | Insert, update, delete, merge |
| `CREATE TABLE`, `CREATE VIEW`, `CREATE FUNCTION`… | Schema | Create objects |
| `EXECUTE` | Function, model | Call it |
| `READ VOLUME` / `WRITE VOLUME` | Volume | File access |
| `MANAGE` | Any | Manage grants and ownership without being owner |
| `ALL PRIVILEGES` | Any | Everything except `MANAGE`-type rights; avoid in prod |

Grants for the ContosoData personas:

```sql
-- Analysts: read gold, discover everything
GRANT USE CATALOG, BROWSE ON CATALOG contoso_prod TO `grp-analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA contoso_prod.gold TO `grp-analysts`;

-- Engineers: build in dev only
GRANT USE CATALOG, CREATE SCHEMA ON CATALOG contoso_dev TO `grp-data-engineers`;

-- Prod writes only through the CI/CD service principal
GRANT USE CATALOG ON CATALOG contoso_prod TO `sp-cicd-prod`;
GRANT USE SCHEMA, CREATE TABLE, MODIFY, SELECT ON SCHEMA contoso_prod.gold TO `sp-cicd-prod`;

-- Review
SHOW GRANTS ON SCHEMA contoso_prod.gold;
SELECT * FROM system.information_schema.table_privileges WHERE table_schema = 'gold';
```

### 8.4 Workspace binding

By default, a catalog is visible from every workspace attached to the metastore. **Bind** prod catalogs to prod workspaces so a dev notebook can never read prod data, even if the user has grants:

```bash
databricks catalogs update contoso_prod --isolation-mode ISOLATED
databricks workspace-bindings update-bindings catalog contoso_prod \
  --json '{"add":[{"workspace_id": 1234567890, "binding_type": "BINDING_TYPE_READ_WRITE"}]}'
```

You can also bind a catalog read-only to an analytics workspace.

### 8.5 Tags, comments and classification

```sql
COMMENT ON TABLE contoso_prod.gold.sales IS 'One row per sale line. Grain: sale_id. Refreshed daily 05:00 UTC. Owner: grp-sales-data-owners.';
ALTER TABLE contoso_prod.gold.sales SET TAGS ('domain' = 'sales', 'tier' = 'gold', 'exposed_via' = 'client-api');
ALTER TABLE contoso_prod.gold.sales ALTER COLUMN customer_email SET TAGS ('pii' = 'email');

-- Find every PII column in the metastore
SELECT catalog_name, schema_name, table_name, column_name, tag_value
FROM system.information_schema.column_tags
WHERE tag_name = 'pii';
```

- **Governed tags** (account-level tag policies) restrict which tag keys and values exist and who can apply them, so tags become trustworthy enough to drive ABAC policies (Module 9).
- **Data classification** can scan tables and suggest PII tags automatically. **AI-generated comments** draft descriptions for tables and columns. Both need human review.
- Use tags such as `certified = true` or `deprecated = true` to signal trust in the absence of (or in addition to) native certification badges, where your release offers them.

### 8.6 Lineage and discovery

- **Lineage** is captured automatically, down to column level, for queries, notebooks, jobs, pipelines and dashboards on Unity Catalog. See the **Lineage** tab, or query `system.access.table_lineage` and `system.access.column_lineage`.
- **Impact analysis:** before changing `gold.sales`, list downstream objects:

```sql
SELECT DISTINCT target_table_full_name, entity_type
FROM system.access.table_lineage
WHERE source_table_full_name = 'contoso_prod.gold.sales'
  AND event_date >= current_date() - 90;
```

- **Discover / Catalog Explorer** lets users search by name, comment and tag. With `BROWSE` they can see objects they can't read and **request access**, routed to owners.

### 8.7 Lab

1. Create `grp-sales-data-owners` and transfer ownership of `contoso_prod` to it.
2. Bind `contoso_prod` to the prod workspace only, and confirm the dev workspace can't see it.
3. Tag `customer_email` as PII and list all PII columns.
4. Run the lineage query and draw the downstream graph of `gold.sales`.

## Module 9 — Fine-grained access: rows, columns, attributes, files

Unity Catalog has three generations of fine-grained control. **Dynamic views** came first, **row filters and column masks** are attached directly to tables, and **ABAC policies** apply filters and masks automatically wherever a governed tag appears. Use the newest one your release supports, because it scales best.

### 9.1 Row filters

A row filter is a SQL UDF returning `BOOLEAN`, bound to one or more columns of a table. Every query from any engine applies it.

```sql
-- Group-based variant: each client has an account group grp-client-<id>
CREATE OR REPLACE FUNCTION contoso_prod.sec.rf_client(cid INT)
RETURNS BOOLEAN
RETURN is_account_group_member('grp-data-engineers')
    OR is_account_group_member(concat('grp-client-', CAST(cid AS STRING)));

ALTER TABLE contoso_prod.gold.sales SET ROW FILTER contoso_prod.sec.rf_client ON (client_id);

-- Remove
ALTER TABLE contoso_prod.gold.sales DROP ROW FILTER;
```

The mapping-table variant (the `sec.principal_client` table in Module 7) is easier to administer when you have hundreds of clients.

### 9.2 Column masks

```sql
CREATE OR REPLACE FUNCTION contoso_prod.sec.mask_email(email STRING)
RETURNS STRING
RETURN CASE
  WHEN is_account_group_member('grp-pii-readers') THEN email
  ELSE regexp_replace(email, '^[^@]+', '***')
END;

ALTER TABLE contoso_prod.gold.sales ALTER COLUMN customer_email SET MASK contoso_prod.sec.mask_email;
```

Masks can take extra columns as arguments, for example to unmask only the rows of the caller's own client: `SET MASK f USING COLUMNS (client_id)`.

### 9.3 ABAC policies (attribute-based)

With ABAC you write the rule once at catalog or schema level. It applies to every table whose columns carry a matching **governed tag**, including tables created later.

```sql
-- Mask every column tagged pii=email in the gold schema, for everyone except PII readers
CREATE POLICY mask_pii_email
ON SCHEMA contoso_prod.gold
COMMENT 'Mask emails everywhere unless in grp-pii-readers'
COLUMN MASK contoso_prod.sec.mask_email
TO `account users` EXCEPT `grp-pii-readers`
FOR TABLES
MATCH COLUMNS has_tag_value('pii', 'email') AS email_col
ON COLUMN email_col;

-- Filter rows on every table with a column tagged client_key
CREATE POLICY client_isolation
ON CATALOG contoso_prod
ROW FILTER contoso_prod.sec.rf_client
TO `account users` EXCEPT `grp-data-engineers`
FOR TABLES
MATCH COLUMNS has_tag('client_key') AS cid
USING COLUMNS (cid);
```

ABAC was in preview with syntax still settling. Treat the above as the shape of the feature and check the current `CREATE POLICY` reference before running it.

### 9.4 Dynamic views (legacy, still useful)

```sql
CREATE OR REPLACE VIEW contoso_prod.api.v_sales_secure AS
SELECT sale_id, client_id, sale_date, amount,
       CASE WHEN is_account_group_member('grp-pii-readers') THEN customer_email ELSE NULL END AS customer_email
FROM contoso_prod.gold.sales
WHERE is_account_group_member('grp-data-engineers')
   OR is_account_group_member(concat('grp-client-', CAST(client_id AS STRING)));
```

Use dynamic views for Delta Sharing (with `CURRENT_RECIPIENT()`), and when you need logic that spans several tables.

### 9.5 Which compute enforces it

Row filters, column masks and ABAC are enforced on SQL warehouses, serverless compute and **standard** (shared) access-mode clusters. On **dedicated** (single-user) clusters, queries against filtered tables are routed through serverless filtering, where supported. Older runtimes may refuse to read the table. Check the minimum Databricks Runtime version in the docs.

### 9.6 Files: external locations and volumes

```sql
-- Admin: register cloud storage once
CREATE STORAGE CREDENTIAL sc_landing WITH (AZURE_MANAGED_IDENTITY ...);   -- or AWS IAM role / GCP SA
CREATE EXTERNAL LOCATION el_landing URL 'abfss://landing@contosolake.dfs.core.windows.net/' WITH (STORAGE CREDENTIAL sc_landing);

-- Engineers may read raw files there, nothing else
GRANT READ FILES ON EXTERNAL LOCATION el_landing TO `grp-data-engineers`;

-- Governed file area for a client's uploads
CREATE VOLUME contoso_prod.api.client_uploads;
GRANT READ VOLUME, WRITE VOLUME ON VOLUME contoso_prod.api.client_uploads TO `sp-client-northwind`;
```

Never give users direct cloud-storage access (storage keys, IAM roles) to Unity Catalog–managed data. That bypasses every control in this module.

### 9.7 Lab

1. Apply `rf_client` to `gold.sales`, then query as `sp-client-northwind` and as an engineer.
2. Apply the email mask, and confirm Delta Sharing and the Statement API both return masked values.
3. (If available) replace the table-level mask with an ABAC policy, create a new table with a `pii=email` column, and confirm it's masked automatically.
4. Try to read `gold.sales` from a dedicated cluster on an old runtime and note the error.

## Module 10 — Audit, monitoring and cost governance

Databricks exposes its own operational data as **system tables** in the `system` catalog. Audit, lineage, billing, query history and compute usage can all be queried with SQL, so governance reporting is just another dashboard.

### 10.1 Enable and grant system tables

- Most schemas are on by default in recent releases. Enable any missing ones with the account API or `databricks system-schemas enable <metastore-id> <schema>`.
- System tables contain sensitive data (who queried what). Grant them to a governance group only:

```sql
GRANT USE CATALOG ON CATALOG system TO `grp-governance`;
GRANT USE SCHEMA, SELECT ON SCHEMA system.access TO `grp-governance`;
GRANT USE SCHEMA, SELECT ON SCHEMA system.billing TO `grp-finops`;
```

| Table | Contents |
| --- | --- |
| `system.access.audit` | Every account and workspace event: logins, grants, queries, shares, app calls |
| `system.access.table_lineage` / `column_lineage` | Lineage edges with the job, notebook or query that created them |
| `system.query.history` | Every SQL warehouse and serverless query: text, user, duration, rows, bytes |
| `system.billing.usage` | DBUs by SKU, workspace, warehouse, job, app, tags |
| `system.billing.list_prices` | List prices to convert DBUs into currency |
| `system.compute.warehouses`, `clusters`, `node_timeline` | Compute configuration and utilisation |
| `system.information_schema.*` | Grants, tags, tables, columns across the metastore |

### 10.2 Governance queries you'll reuse

Grant changes in the last 30 days:

```sql
SELECT event_time, user_identity.email AS actor, action_name,
       request_params.securable_full_name AS object, request_params.changes AS changes
FROM system.access.audit
WHERE service_name = 'unityCatalog' AND action_name = 'updatePermissions'
  AND event_date >= current_date() - 30
ORDER BY event_time DESC;
```

API usage per client service principal (rows and time):

```sql
SELECT executed_by AS principal, date_trunc('DAY', start_time) AS day,
       count(*) AS queries, sum(produced_rows) AS rows_returned,
       percentile(total_duration_ms, 0.95) AS p95_ms
FROM system.query.history
WHERE compute.warehouse_id = '<api-warehouse-id>' AND start_time >= current_date() - 7
GROUP BY ALL ORDER BY day DESC, rows_returned DESC;
```

Cost of the client API by day, in list-price dollars:

```sql
SELECT u.usage_date, sum(u.usage_quantity * p.pricing.default) AS usd
FROM system.billing.usage u
JOIN system.billing.list_prices p
  ON u.sku_name = p.sku_name AND u.usage_start_time >= p.price_start_time
 AND (p.price_end_time IS NULL OR u.usage_start_time < p.price_end_time)
WHERE u.custom_tags['cost_center'] = 'client-api' AND u.usage_date >= current_date() - 30
GROUP BY u.usage_date ORDER BY u.usage_date;
```

Column names in system tables evolve. Run `DESCRIBE` on each table before you build alerts on it.

### 10.3 Alerts

Turn the queries into **SQL alerts** (Databricks SQL → Alerts) with a schedule and a condition. Useful examples:

- More than 0 grant changes on `contoso_prod` outside the CI/CD SP.
- One client SP returning more than 5 million rows in a day (possible scraping).
- Daily API cost more than 2× the 30-day average.
- A table tagged `tier = gold` without an owner group, or without a comment.

### 10.4 Data quality monitoring

- **Lakehouse Monitoring / data quality monitoring** attaches to a table and computes profile and drift metrics (null rates, distributions, freshness) into metric tables, with an auto-generated dashboard. Some releases also offer **anomaly detection** for freshness and completeness at schema level.
- **Expectations** in Lakeflow Declarative Pipelines (formerly DLT) enforce rules at write time:

```sql
CREATE OR REFRESH STREAMING TABLE silver_sales (
  CONSTRAINT valid_amount EXPECT (amount >= 0) ON VIOLATION DROP ROW,
  CONSTRAINT has_client  EXPECT (client_id IS NOT NULL) ON VIOLATION FAIL UPDATE
) AS SELECT * FROM STREAM(contoso_prod.bronze.sales_raw);
```

### 10.5 Cost governance

- **Budgets** (account console → Usage → Budgets): monthly targets by workspace or tag, with email alerts.
- **Tag everything**: warehouses, jobs, apps and serverless budget policies carry `cost_center`, `client`, `env` tags into `system.billing.usage`.
- **Serverless budget policies** attach tags to serverless usage that has no cluster to tag, such as notebooks and Apps.
- **Compute policies** (Module 11) cap cluster sizes and enforce auto-termination.
- The **Usage dashboard** (account console) is a ready-made view over the billing tables.

### 10.6 Lab

1. Grant `system.access` to `grp-governance` and run the grant-changes query.
2. Build an AI/BI dashboard "Client API" with queries per client, rows returned, p95 latency and cost per day.
3. Create an alert for any client exceeding 5 million rows per day.
4. Attach a quality monitor to `gold.sales` and review its drift dashboard after two refreshes.

## Module 11 — ALM, identity and network

Production Databricks is changed only by code, accessed only by known identities, and reachable only through approved network paths. This module covers the controls for each.

### 11.1 Databricks Asset Bundles (DABs)

A bundle is a folder with a `databricks.yml` that declares jobs, pipelines, apps, dashboards, schemas, volumes, model-serving endpoints and their permissions, with one **target** per environment.

```yaml
# databricks.yml
bundle:
  name: contoso_client_api

variables:
  catalog: { default: contoso_dev }
  warehouse_id: { description: SQL warehouse for the API }

resources:
  apps:
    client_api:
      name: client-api-${bundle.target}
      source_code_path: ./app
      resources:
        - name: sql-warehouse
          sql_warehouse: { id: ${var.warehouse_id}, permission: CAN_USE }
      permissions:
        - group_name: grp-api-callers
          level: CAN_USE
  jobs:
    refresh_gold:
      name: refresh-gold-${bundle.target}
      tasks:
        - task_key: build
          notebook_task: { notebook_path: ./src/build_gold.py }
      schedule: { quartz_cron_expression: "0 0 5 * * ?", timezone_id: UTC }

targets:
  dev:
    mode: development
    default: true
    workspace: { host: https://adb-111.azuredatabricks.net }
  prod:
    mode: production
    workspace: { host: https://adb-222.azuredatabricks.net }
    variables: { catalog: contoso_prod, warehouse_id: "abcdef1234567890" }
    run_as: { service_principal_name: "<sp-cicd-prod-app-id>" }
```

```bash
databricks bundle validate -t prod
databricks bundle deploy  -t prod      # run from CI only
```

`mode: production` refuses deploys from a local laptop's uncommitted state, and `run_as` makes prod jobs run as the CI/CD SP, not a person.

### 11.2 Git folders and CI/CD

- **Git folders** link workspace folders to a Git branch for development.
- Production deploys run from **GitHub Actions or Azure DevOps** with `databricks bundle deploy`, authenticated by **OAuth token federation** (GitHub OIDC → Databricks), so no secret is stored in CI.
- Branch protection plus required reviews on `main` become your change-approval control. Audit logs show the SP as the actor for every prod change.
- **Terraform** (`databricks/databricks` provider) manages account-level objects: metastores, catalogs, grants, workspaces, network configs. Keep it in a separate platform repo.

### 11.3 Identity

| Control | Configure in | Recommendation |
| --- | --- | --- |
| **SSO** | Account console → Settings | Mandatory; MFA enforced by your IdP |
| **SCIM provisioning** / automatic identity management | Account console | Sync users and groups from Entra ID / Okta; no manual users |
| **Account groups** | Account console | Grant only to account groups, not workspace-local groups |
| **Entitlements** | Workspace admin | Remove *Workspace access* or *Databricks SQL access* where not needed; restrict *Allow cluster creation* |
| **PATs** | Workspace admin → Advanced | Disable for users, or restrict with token permissions and a max lifetime |
| **Service principals** | Account console | One per integration or client; owner group; OAuth secret expiry tracked |
| **Admin roles** | Account and workspace | Few admins; use your IdP's PIM / JIT group membership |

### 11.4 Compute policies

Compute policies restrict which clusters users can create:

```json
{
  "autotermination_minutes": { "type": "range", "maxValue": 60, "defaultValue": 30 },
  "num_workers":             { "type": "range", "maxValue": 8 },
  "data_security_mode":      { "type": "fixed", "value": "USER_ISOLATION" },
  "custom_tags.cost_center": { "type": "fixed", "value": "data-eng" }
}
```

This policy forces standard access mode (so row filters and masks are enforced), caps cluster size, forces auto-termination and adds a cost tag.

### 11.5 Network

| Control | What it does |
| --- | --- |
| **IP access lists** (account console and workspace) | Allow UI/API access only from listed IP ranges |
| **Private Link / Private Service Connect** — front-end | Users and clients reach the workspace through a private endpoint |
| **Private Link** — back-end | Classic compute talks to the control plane privately |
| **VNet / VPC injection** (customer-managed network) | Classic clusters run in your network with your firewall and NSGs |
| **Serverless network policies / egress control** | Restrict which internet or cloud destinations serverless compute, Apps and model serving can reach (anti-exfiltration) |
| **Network connectivity configurations (NCC)** | Stable private endpoints from serverless compute to your storage and services |
| **Storage firewalls** | Lock the cloud storage of managed catalogs to Databricks access connectors only |

### 11.6 Encryption and compliance

- **Customer-managed keys** for managed services (notebooks, queries) and for workspace storage, using your KMS or Key Vault.
- The **Compliance security profile** and **enhanced security monitoring** add hardened images, monitoring agents and controls for HIPAA, PCI-DSS, FedRAMP and similar regimes.
- **Data residency:** choose the workspace region. The metastore is regional, and cross-region access goes through Delta Sharing.

### 11.7 Lab

1. Convert Modules 4 and 6 into a bundle with `dev` and `prod` targets.
2. Configure GitHub OIDC federation for `sp-cicd-prod` and deploy from a GitHub Action.
3. Create the compute policy above and assign it to `grp-data-engineers`.
4. Add a workspace IP access list that allows only office and gateway IPs, and test from a phone hotspot (it should be denied).

## Module 12 — Capstone, quiz and cheat sheet

### 12.1 Capstone project

**Brief:** onboard two new clients in one day. *Tailspin* wants a REST API with a key; *Woodgrove* runs Databricks and wants the dataset in its own Unity Catalog.

- [ ] Tailspin: key issued in the gateway, mapped in `sec.api_subscription`, served by the external FastAPI (Module 6)
- [ ] Woodgrove: Databricks-to-Databricks recipient with property `client_id`, granted `client_sales` (Module 5)
- [ ] Row filter or ABAC policy proves isolation for both
- [ ] Email column masked for both
- [ ] Everything deployed through a bundle from CI with a federated SP
- [ ] AI/BI dashboard showing both clients' usage and cost from system tables
- [ ] Alert for abnormal row volumes per client
- [ ] One-page data-sharing record per client: data, purpose, retention, owner, review date

**Assessment:** the reviewer tries to read another client's rows (API and share), use an expired token, deploy to prod from a laptop, and read the raw storage path. All four must fail.

### 12.2 Knowledge check

1. Which Databricks option needs no API server at all to give a client live tables?
2. What does `CURRENT_RECIPIENT('client_id')` return, and where can you use it?
3. Why prefer OAuth M2M over personal access tokens for integrations?
4. In Databricks Apps, what's the difference between app authorization and user authorization?
5. Which Statement Execution API setting do you use for results larger than about 25 MiB?
6. What does binding a catalog to a workspace prevent?
7. Which privilege lets users see an object's metadata without reading data?
8. Name the system table you'd query for (a) grant changes, (b) rows returned per client, (c) cost.
9. Why does a metric view matter for an API that returns ratios?
10. On which compute types are row filters and column masks enforced?

**Answers**

1. Delta Sharing.
2. The value of the recipient's property. Use it in views added to a share, to filter rows per recipient.
3. Short-lived tokens, not tied to a person, can be scoped, rotate without downtime, and support federation (no secret at all).
4. App authorization queries as the app's SP. User authorization queries as the caller (on-behalf-of), so Unity Catalog applies the caller's grants and filters.
5. `disposition: EXTERNAL_LINKS` (with `ARROW_STREAM` or `CSV`).
6. Access to that catalog from any non-bound workspace, even by users who hold grants.
7. `BROWSE`.
8. (a) `system.access.audit`, (b) `system.query.history`, (c) `system.billing.usage` joined to `list_prices`.
9. It recomputes measures at the requested grain, so ratios like average basket are correct for any grouping instead of averaging averages.
10. SQL warehouses, serverless compute and standard-access-mode clusters. Dedicated clusters need serverless filtering support or a recent runtime.

### 12.3 Cheat sheet

| Need | Use |
| --- | --- |
| Client wants whole tables, live | Delta Sharing (+ `CURRENT_RECIPIENT()` dynamic view) |
| Simple REST SQL access | Statement Execution API with OAuth M2M |
| Custom endpoints without extra infra | Databricks Apps (user authorization for per-caller security) |
| Paid API product with keys and quotas | External FastAPI + APIM / AWS API Gateway / Apigee |
| Millisecond lookups, high QPS | Lakebase synced tables behind an API |
| Same KPIs everywhere | Metric views + `MEASURE()` |
| Hide rows | Row filters, ABAC row policies, dynamic views |
| Hide columns | Don't expose → column masks / ABAC masks |
| Isolate environments | Catalog per env + workspace binding |
| Who did what | `system.access.audit` |
| Lineage and impact | Lineage tab, `system.access.table_lineage` |
| Cost | `system.billing.usage`, budgets, tags, serverless budget policies |
| Safe change | Asset Bundles + CI/CD + token federation |
| Private access | IP access lists, Private Link, serverless egress policies |

### Further reading

Check each feature's status in your cloud before you deliver this course. The links below follow the usual Databricks docs paths but weren't individually checked for this draft.

- [Statement Execution API](https://docs.databricks.com/aws/en/dev-tools/sql-execution-tutorial)
- [Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Delta Sharing](https://docs.databricks.com/aws/en/delta-sharing/)
- [Unity Catalog privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges)
- [Row filters and column masks](https://docs.databricks.com/aws/en/tables/row-and-column-filters)
- [Metric views](https://docs.databricks.com/aws/en/metric-views/)
- [System tables](https://docs.databricks.com/aws/en/admin/system-tables/)
- [Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [OAuth M2M](https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m)
