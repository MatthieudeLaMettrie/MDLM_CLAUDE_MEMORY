# Microsoft Fabric — Data APIs & Governance: Full Course

> Source of truth: the live Claude Doc (https://claude.ai/code/artifact/d0201f53-da3b-4837-9af2-15bc399d7c73), which also holds the drawn diagrams. Exported 2026-10-13. Written from model knowledge; preview-era syntax is flagged in the text for verification.


Oct 13, 2026 · @Matt

## Course overview

By the end of this course you can expose Fabric Warehouse and semantic-model data to external clients through a secure API, and govern the whole estate with Fabric, Purview and Entra ID controls.

**Audience:** data engineers, analytics engineers and BI leads who already know SQL and basic Python and have built at least one Fabric Lakehouse or Warehouse.

**Prerequisites:** a Fabric capacity (trial or F2+), workspace Admin rights, an Entra ID tenant where you can register an app, and an Azure subscription for Modules 5–6.

**Running scenario.** You run *ContosoData*, a company that sells sales analytics to retail clients. Each client must pull its own data — and only its own — from your Gold Warehouse and from the `Sales Analytics` semantic model, straight into the client's own systems. Every module builds one piece of that solution.

### Learning outcomes

1. Explain how tenant, capacity, workspace, item and OneLake fit together, and where security is enforced.
2. Choose between GraphQL, User Data Functions, a custom FastAPI and a direct connection for a given client need.
3. Build and call each API option with service-principal authentication.
4. Isolate clients with row-level security so one client never sees another's rows.
5. Design a governance model with domains, endorsement, sensitivity labels, DLP, audit and monitoring.
6. Ship changes safely with Git, deployment pipelines and network controls.

### Module map

| # | Module | Type | Time |
| --- | --- | --- | --- |
| 1 | Architecture essentials | Concepts | 45 min |
| 2 | API options compared | Concepts | 30 min |
| 3 | GraphQL API | Lab | 60 min |
| 4 | User Data Functions | Lab | 60 min |
| 5 | Custom FastAPI + API Management | Lab | 2 h |
| 6 | Securing the API | Concepts + lab | 60 min |
| 7 | Governance foundations | Concepts + lab | 45 min |
| 8 | Data access security | Lab | 90 min |
| 9 | Microsoft Purview | Concepts + lab | 60 min |
| 10 | Admin, monitoring, capacity | Concepts + lab | 45 min |
| 11 | ALM and network | Concepts + lab | 60 min |
| 12 | Capstone + quiz | Project | 3 h |

Times are estimates. Fabric ships features monthly, so check feature names and preview status against Microsoft Learn before teaching this material.

## Module 1 — Fabric architecture essentials

Every API and governance decision in Fabric comes down to five levels: tenant → capacity → workspace → item → data inside the item. Each level has its own controls.

### 1.1 The hierarchy

| Level | What it is | Who controls it | Key controls |
| --- | --- | --- | --- |
| **Tenant** | Your organisation's single Fabric/Power BI instance, tied to one Entra ID tenant | Fabric Administrator | Tenant settings, Private Link, audit, sensitivity-label enforcement |
| **Capacity** | The paid compute pool (F2…F2048), measured in Capacity Units (CU) | Capacity admin | Which workspaces run on it, surge protection, region |
| **Domain** | A logical grouping of workspaces by business area | Domain admin | Delegated tenant settings, default label, domain contributors |
| **Workspace** | A container of items, the main security and ALM boundary | Workspace Admin | Roles, Git connection, workspace identity, network rules |
| **Item** | Lakehouse, Warehouse, semantic model, report, notebook, pipeline, GraphQL API, UDF… | Item owner | Item permissions, endorsement, sensitivity label |
| **Data in the item** | Tables, folders, columns, rows | Item owner / data steward | OneLake security, SQL GRANT, RLS/CLS/DDM, semantic-model RLS/OLS |

### 1.2 OneLake: one copy of data, many engines

OneLake is the single data lake behind every workspace. Lakehouse and Warehouse tables are stored as **Delta Parquet** files in it. Several engines read the same files:

- **Spark** (notebooks, jobs) reads and writes the Delta files directly.
- The **SQL analytics endpoint** gives every Lakehouse a read-only T-SQL surface.
- A **Warehouse** is a full read/write T-SQL engine (transactions, `INSERT/UPDATE/MERGE`).
- A **semantic model in Direct Lake mode** loads the same Delta files straight into the VertiPaq engine, with no import refresh.

> **Why this matters for APIs:** any client-facing API you build in this course ends up querying one of three surfaces — the SQL endpoint/Warehouse (T-SQL over TDS), the semantic model (DAX via REST or XMLA), or GraphQL (which itself compiles to SQL against the Warehouse).

### 1.3 Connection surfaces you will use

| Surface | Protocol | Address format | Typical client |
| --- | --- | --- | --- |
| Warehouse / SQL endpoint | TDS (port 1433) | `<id>.datawarehouse.fabric.microsoft.com`, database = item name | ODBC, JDBC, pyodbc, SSMS |
| Semantic model REST | HTTPS | `api.powerbi.com/v1.0/myorg/groups/{ws}/datasets/{id}/executeQueries` | Any HTTP client |
| Semantic model XMLA | XMLA over HTTPS | `powerbi://api.powerbi.com/v1.0/myorg/<Workspace>` | ADOMD.NET, Tabular Editor, DAX Studio |
| GraphQL API | HTTPS POST | `Copied from the API item's Copy endpoint button` | Any HTTP/GraphQL client |
| User Data Functions | HTTPS POST | Endpoint shown on the function's *Functions* pane | Any HTTP client |
| Fabric REST API | HTTPS | `api.fabric.microsoft.com/v1/...` | Automation, admin scripts |

Find the SQL connection string under the Warehouse's **Settings → SQL endpoint**, or with the **Copy SQL connection string** button.

### 1.4 Identity: Entra ID everywhere

Fabric has no local users and no API keys of its own. Every call carries an **Entra ID OAuth 2.0 token**. There are three identity types:

| Identity | Use it for | Notes |
| --- | --- | --- |
| **User** (member or B2B guest) | Interactive tools, client analysts | MFA and Conditional Access apply |
| **Service principal** (app registration) | Server-to-server APIs, CI/CD | Must be allowed by the tenant setting *Service principals can use Fabric APIs* |
| **Managed identity / workspace identity** | Azure resources calling Fabric (Container Apps, Functions); Fabric reaching firewalled storage | No secret to rotate — prefer it whenever possible |

Token **scopes (audiences)** you will request:

- Fabric REST and GraphQL: `https://api.fabric.microsoft.com/.default`
- Power BI REST (executeQueries): `https://analysis.windows.net/powerbi/api/.default`
- SQL endpoint (when passing a raw token to ODBC): `https://database.windows.net/.default`

### 1.5 Exercise

1. In a trial capacity, create workspace `contoso-gold-dev`.
2. Create Warehouse `wh_gold` and run the setup script below.
3. Copy its SQL connection string and connect from SSMS or VS Code with your own account.

```sql
CREATE SCHEMA core;
GO
CREATE TABLE core.client (client_id INT NOT NULL, client_name VARCHAR(100) NOT NULL, tenant_upn_domain VARCHAR(100) NOT NULL);
CREATE TABLE core.sales (sale_id BIGINT NOT NULL, client_id INT NOT NULL, sale_date DATE NOT NULL, store VARCHAR(50), product VARCHAR(100), qty INT, amount DECIMAL(18,2), customer_email VARCHAR(200));
INSERT INTO core.client VALUES (1,'Northwind','northwind.com'),(2,'Fabrikam','fabrikam.com');
INSERT INTO core.sales VALUES
 (1,1,'2026-09-01','Paris-01','Coffee',3,12.60,'a@x.com'),
 (2,1,'2026-09-02','Paris-02','Tea',1,4.20,'b@x.com'),
 (3,2,'2026-09-01','Lyon-01','Coffee',5,21.00,'c@y.com');
```

Note that Fabric Warehouse doesn't enforce `PRIMARY KEY` or `FOREIGN KEY` constraints, so keys are declared `NOT ENFORCED` if you declare them at all. This setup data is reused in every lab.

## Module 2 — API options compared

Fabric gives you two native API features (GraphQL and User Data Functions) and two "bring your own" routes (a custom web API, or a direct database connection). External, paying clients usually need the custom route, because only it gives you API keys, quotas and a versioned REST contract.

### 2.1 Side-by-side comparison

| Criterion | API for GraphQL | User Data Functions | Custom FastAPI + APIM | Direct SQL / XMLA |
| --- | --- | --- | --- | --- |
| Hosted in | Fabric | Fabric | Azure (your subscription) | n/a |
| Protocol | GraphQL over HTTPS | REST (POST, JSON) | REST, any shape you design | TDS / XMLA |
| Reads Warehouse / Lakehouse | Yes | Yes | Yes | Yes |
| Reads semantic model | No | Indirectly (call the REST API from code) | Yes (executeQueries or XMLA) | Yes (XMLA) |
| Writes data | Yes (mutations, on Warehouse / SQL DB) | Yes | Yes | Yes |
| Client auth | Entra token | Entra token | API key, OAuth, mTLS — your choice | Entra identity |
| Rate limiting, quotas | No | No | Yes (APIM policies) | No |
| OpenAPI / developer portal | Schema introspection | No | Yes | No |
| Compute cost | Fabric CU | Fabric CU | Azure compute + Fabric CU for queries | Fabric CU |
| Build effort | Hours | Hours–days | Days–weeks | Minutes |
| Best for | Internal apps, partners in your tenant | Custom logic, light write-back | Commercial client APIs | Technical clients, BI tools |

### 2.2 What each option is, in one paragraph

**API for GraphQL** is a Fabric item that reads the schema of a data source and generates a GraphQL schema with queries (with filter, sort and pagination) and mutations. No code is needed. The caller's identity flows through to the source, so Warehouse RLS applies.

**User Data Functions (UDF)** are Python functions you write in a Fabric item. Each function marked `@udf.function()` becomes an HTTPS endpoint. They can use managed connections to Warehouse, Lakehouse and SQL database, plus any PyPI library you add.

**Custom FastAPI** is a normal Python web app you deploy to Azure Container Apps, App Service or Functions. It authenticates to Fabric as a managed identity or service principal. **Azure API Management (APIM)** in front adds subscription keys, quotas, caching, a developer portal and versioning.

**Direct connection** gives the client a service principal (or B2B guest user) with read access to a dedicated schema. The client queries with its own tools. It's the cheapest option, but you have no contract, quota or usage metering.

### 2.3 Decision tree

&#91;embedded content: API option decision tree · 4 questions, 5 outcomes\]

Answer the questions top to bottom and stop at the first "yes". You can combine options: GraphQL for internal apps and the custom API for clients, both reading the same `api` views.

### 2.4 Common mistakes

- **Exposing base tables.** Always expose views in a dedicated `api` schema, so you can refactor the tables underneath.
- **One service principal shared by all clients.** You lose per-client audit and can't revoke one client. Use one identity per client, or a gateway that maps keys to clients.
- **Using notebooks as APIs.** Notebooks can be triggered through the Job Scheduler REST API, but they start in seconds-to-minutes and return no synchronous result. They are not request/response APIs.
- **Bulk extracts through executeQueries.** It's capped at about 100,000 rows or 1,000,000 values per query. For bulk data, serve files (for example a Delta Sharing–style export or a pre-signed ADLS copy) or SQL.
- **Forgetting capacity cost.** Every API query burns CU on your capacity. A noisy client can throttle your internal reports.

## Module 3 — Lab: Fabric API for GraphQL

In this lab a Python client in another system reads its own sales from `wh_gold` through a GraphQL endpoint, authenticated as a service principal. You write no server code.

### 3.1 Prepare an API-facing view

Never expose `core.*` tables. Create a contract schema:

```sql
CREATE SCHEMA api;
GO
CREATE VIEW api.v_sales AS
SELECT s.sale_id, s.client_id, s.sale_date, s.store, s.product, s.qty, s.amount
FROM core.sales AS s;   -- customer_email deliberately NOT exposed
```

### 3.2 Create the GraphQL item

1. In `contoso-gold-dev`, choose **New item → API for GraphQL** and name it `gql_client_sales`.
2. Choose **Select data source**, then the connectivity option:
   - **Single sign-on (SSO):** the caller's own identity runs the query. Warehouse permissions and RLS apply per caller. Use this for client isolation.
   - **Saved credentials:** every caller runs as one stored identity. It's simpler, but RLS sees only that identity.
3. Pick `wh_gold` and select only `api.v_sales`.
4. Fabric generates the schema. In the **Schema explorer** you see a type for the view, a query that lists rows and, for tables with keys, single-item queries and mutations.
5. Optional: in **Manage relationships**, link types (for example client → sales, one-to-many) so callers can fetch nested data in one request.
6. Optional: expose a Warehouse **stored procedure** as a mutation for controlled write-back.

### 3.3 Test in the editor

The built-in editor runs queries with your own identity:

```graphql
query SalesSince($from: Date!) {
  v_sales(
    first: 100
    filter: { sale_date: { gte: $from } }
    orderBy: { sale_date: DESC }
  ) {
    items { sale_id client_id sale_date store product qty amount }
    endCursor
    hasNextPage
  }
}
```

Variables: `{ "from": "2026-09-01" }`. Pagination is cursor-based: pass `after: "<endCursor>"` to get the next page while `hasNextPage` is true. The exact generated query names depend on your object names. Always check the Schema explorer instead of guessing them.

### 3.4 Give a client a service principal

1. **Entra ID → App registrations → New registration:** create `sp-client-northwind` and add a client secret, or better a certificate.
2. Put the service principal in an Entra security group `sg-fabric-api-clients`.
3. **Admin portal → Tenant settings → Developer settings:** enable *Service principals can use Fabric APIs* for `sg-fabric-api-clients` only.
4. On the GraphQL item, **Manage permissions → Add**: grant the service principal **Run Queries and Mutations**.
5. Because the API uses SSO, also grant the service principal read access on the data. Share `wh_gold` with **Read** only (no ReadData), then `GRANT SELECT ON SCHEMA::api TO [sp-client-northwind];`.

### 3.5 Call it from the client's system (Python)

```python
import requests
from azure.identity import ClientSecretCredential

TENANT = "<your-tenant-id>"
CLIENT_ID = "<sp-client-northwind app id>"
SECRET = "<secret from your vault>"
ENDPOINT = "<copied from the GraphQL item: Copy endpoint>"

cred = ClientSecretCredential(TENANT, CLIENT_ID, SECRET)
token = cred.get_token("https://api.fabric.microsoft.com/.default").token

query = """
query($from: Date!, $after: String) {
  v_sales(first: 500, after: $after, filter: { sale_date: { gte: $from } }) {
    items { sale_id sale_date store product qty amount }
    endCursor hasNextPage
  }
}"""

rows, after = [], None
while True:
    r = requests.post(ENDPOINT,
        headers={"Authorization": f"Bearer {token}"},
        json={"query": query, "variables": {"from": "2026-09-01", "after": after}},
        timeout=60)
    r.raise_for_status()
    page = r.json()["data"]["v_sales"]
    rows += page["items"]
    if not page["hasNextPage"]:
        break
    after = page["endCursor"]

print(len(rows), "rows")
```

The same call works from curl, Postman, Power Automate, C# (`HttpClient`) or JavaScript `fetch`. Only the token acquisition changes.

### 3.6 Clients in another Entra tenant

- **Option A — you own the identity:** create the service principal in *your* tenant and hand the client its credentials, ideally a certificate exchanged securely. It's simple, and you can revoke it.
- **Option B — B2B guest:** invite the client's user as a guest. They sign in with their own company account under your Conditional Access policies. This is good for analysts, awkward for machines.
- **Option C — multi-tenant app:** the client's app registration is consented into your tenant. It's the most work, but the client manages its own secret.

### 3.7 Lab check

- [ ] The query returns rows in the editor.
- [ ] The Python script returns rows as the service principal.
- [ ] `customer_email` isn't in the schema.
- [ ] Removing the service principal's GraphQL permission makes the script fail with 401/403.

**Limits to know:** GraphQL can't read semantic models. Every request consumes CU on your capacity. There's no built-in per-client throttling, so for paying clients put APIM in front (Module 5) or move to the custom API.

## Module 4 — Lab: User Data Functions as REST endpoints

User Data Functions (UDF) let you write Python business logic inside Fabric and call each function over HTTPS. Pick them over GraphQL when you need validation, computed outputs, calls to other services or controlled write-back.

### 4.1 How UDF works

- One **User data functions** item holds one or more Python functions in a `function_app.py`-style file.
- Each function decorated with `@udf.function()` gets its own invocation URL.
- **Connections** (`@udf.connection`) inject a ready-made client for a Fabric Warehouse, SQL database, Lakehouse (files and SQL endpoint) or Variable library. No credentials appear in code.
- **Libraries:** add PyPI packages in **Library management** (for example `pandas`).
- **Context:** an optional `UserDataFunctionContext` argument tells you who called the function (the invoking identity).
- **Cost and limits:** functions run on your capacity's CU. Runs have a timeout and response-size limits. Check the current values in Microsoft Learn before you design around them.

### 4.2 Create the item

1. **New item → User data functions**, name `udf_client_api`.
2. **Manage connections → Add data connection →** pick `wh_gold`. Note its **alias**, for example `whgold`.
3. In **Library management**, keep the default `fabric-user-data-functions` package.

### 4.3 Identity-to-client mapping table

Never trust a `client_id` that the caller sends. Map the caller's identity to a client instead:

```sql
CREATE TABLE api.client_identity (
  principal_object_id VARCHAR(64) NOT NULL,   -- Entra object id of the SP or user
  client_id INT NOT NULL
);
INSERT INTO api.client_identity VALUES ('<object-id-of-sp-client-northwind>', 1);
```

### 4.4 The function code

```python
import datetime
import fabric.functions as fn

udf = fn.UserDataFunctions()

@udf.connection(argName="wh", alias="whgold")
@udf.context(argName="ctx")
@udf.function()
def get_sales(wh: fn.FabricSqlConnection, ctx: fn.UserDataFunctionContext,
              from_date: str, to_date: str) -> list:
    # 1. validate input
    d_from = datetime.date.fromisoformat(from_date)
    d_to = datetime.date.fromisoformat(to_date)
    if (d_to - d_from).days > 366:
        raise fn.UserThrownError("Date range is limited to 366 days.",
                                 {"from_date": from_date, "to_date": to_date})

    # 2. who is calling?
    caller_oid = ctx.executing_user["Oid"]

    conn = wh.connect()
    cur = conn.cursor()
    # 3. resolve the client from identity, never from a parameter
    cur.execute("SELECT client_id FROM api.client_identity WHERE principal_object_id = ?", caller_oid)
    row = cur.fetchone()
    if row is None:
        raise fn.UserThrownError("Caller is not mapped to a client.")
    client_id = row[0]

    # 4. parameterised query (no string concatenation, no SQL injection)
    cur.execute("""
        SELECT sale_id, sale_date, store, product, qty, amount
        FROM api.v_sales
        WHERE client_id = ? AND sale_date BETWEEN ? AND ?
        ORDER BY sale_date""", client_id, d_from, d_to)
    cols = [c[0] for c in cur.description]
    result = [dict(zip(cols, [str(v) if isinstance(v, (datetime.date,)) else float(v) if hasattr(v, "as_tuple") else v for v in r])) for r in cur.fetchall()]
    cur.close(); conn.close()
    return result
```

**Read the code in four steps:** validate, identify the caller, resolve the client from identity, then run a parameterised query. Return types must be JSON-serialisable, which is why dates and decimals are converted. Check the attribute names of the context object (here `executing_user["Oid"]`) against the current UDF documentation for your release.

### 4.5 Publish, permission, invoke

1. Choose **Publish**. In **Functions explorer**, open `get_sales` and choose **Copy function URL**.
2. In the item's **Manage permissions**, grant `sp-client-northwind` **Execute**.
3. Call it from the client:

```python
import requests
from azure.identity import ClientSecretCredential

cred = ClientSecretCredential(TENANT, CLIENT_ID, SECRET)
token = cred.get_token("https://api.fabric.microsoft.com/.default").token

r = requests.post("<copied function URL>",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json={"from_date": "2026-09-01", "to_date": "2026-09-30"}, timeout=120)
r.raise_for_status()
print(r.json())
```

The request body is a JSON object whose keys are the function's parameter names. The response wraps your return value with status and log metadata.

### 4.6 Write-back example

You can use a function to let clients submit corrections into a staging table you review:

```python
@udf.connection(argName="wh", alias="whgold")
@udf.context(argName="ctx")
@udf.function()
def submit_correction(wh: fn.FabricSqlConnection, ctx: fn.UserDataFunctionContext,
                      sale_id: int, new_qty: int, reason: str) -> str:
    if new_qty < 0 or len(reason) > 500:
        raise fn.UserThrownError("Invalid correction.")
    conn = wh.connect(); cur = conn.cursor()
    cur.execute("""INSERT INTO api.correction_request (sale_id, new_qty, reason, requested_by, requested_at)
                   VALUES (?, ?, ?, ?, SYSUTCDATETIME())""", sale_id, new_qty, reason, ctx.executing_user["Oid"])
    conn.commit(); cur.close(); conn.close()
    return "queued"
```

### 4.7 Lab check

- [ ] The function test pane returns rows for your own mapped identity.
- [ ] The service principal gets only client 1's rows, even when it tries to pass another client id.
- [ ] A date range over 366 days returns your error message, not a stack trace.
- [ ] The invocation appears in the item's **logs / monitoring** view.

## Module 5 — Lab: custom FastAPI behind API Management

This is the production pattern for selling data to external clients. Azure API Management (APIM) owns client keys, quotas and the developer portal. A FastAPI app on Azure Container Apps owns the business logic. Fabric serves the data through the Warehouse (T-SQL) and the semantic model (DAX).

&#91;embedded content: Client API architecture · APIM, FastAPI, Fabric, Key Vault, Entra ID\]

The client's key stops at APIM. Only the FastAPI app (highlighted) decides which client's rows leave Fabric, and it reaches Fabric with a managed identity, not a stored password.

### 5.1 Project layout

```
client-data-api/
├── app/
│   ├── main.py          # FastAPI routes
│   ├── config.py        # settings from env vars
│   ├── security.py      # trust the APIM gateway, resolve client
│   ├── warehouse.py     # T-SQL access
│   └── semantic.py      # DAX via executeQueries
├── requirements.txt
└── Dockerfile
```

`requirements.txt`:

```
fastapi==0.115.*
uvicorn[standard]==0.30.*
pyodbc==5.1.*
azure-identity==1.17.*
requests==2.32.*
pydantic-settings==2.4.*
```

### 5.2 Configuration

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    sql_server: str            # xxxx.datawarehouse.fabric.microsoft.com
    sql_database: str = "wh_gold"
    pbi_workspace_id: str
    pbi_dataset_id: str
    gateway_secret: str        # shared secret only APIM knows (from Key Vault)
    azure_client_id: str | None = None   # user-assigned managed identity, if used

settings = Settings()
```

### 5.3 Warehouse access with managed identity

```python
# app/warehouse.py
import pyodbc
from .config import settings

pyodbc.pooling = True

def _conn_str() -> str:
    s = ("Driver={ODBC Driver 18 for SQL Server};"
         f"Server={settings.sql_server},1433;Database={settings.sql_database};"
         "Encrypt=yes;TrustServerCertificate=no;"
         "Authentication=ActiveDirectoryMsi;")
    if settings.azure_client_id:              # user-assigned identity
        s += f"UID={settings.azure_client_id};"
    return s

def query(sql: str, *params) -> list[dict]:
    with pyodbc.connect(_conn_str(), timeout=30) as cn:
        cur = cn.cursor()
        cur.execute(sql, *params)
        cols = [c[0] for c in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]
```

`ActiveDirectoryMsi` makes the ODBC driver fetch a token for the Container App's managed identity, so there's no secret in code or config. Add that identity to the workspace or Warehouse the way you would add a service principal (Module 6).

### 5.4 Semantic model access (DAX)

```python
# app/semantic.py
import requests
from azure.identity import DefaultAzureCredential
from .config import settings

_cred = DefaultAzureCredential()
PBI_SCOPE = "https://analysis.windows.net/powerbi/api/.default"

def run_dax(dax: str) -> list[dict]:
    token = _cred.get_token(PBI_SCOPE).token          # cached + refreshed by azure-identity
    url = (f"https://api.powerbi.com/v1.0/myorg/groups/{settings.pbi_workspace_id}"
           f"/datasets/{settings.pbi_dataset_id}/executeQueries")
    r = requests.post(url, headers={"Authorization": f"Bearer {token}"},
                      json={"queries": [{"query": dax}],
                            "serializerSettings": {"includeNulls": True}}, timeout=120)
    r.raise_for_status()
    rows = r.json()["results"][0]["tables"][0]["rows"]
    # keys come back as 'Table[Column]' or '[Measure]' — clean them
    return [{k.split("[")[-1].rstrip("]"): v for k, v in row.items()} for row in rows]
```

### 5.5 Trusting the gateway and resolving the client

APIM authenticates the client with its subscription key, then forwards two headers: a shared secret proving the call came through APIM, and the subscription ID identifying the client.

```python
# app/security.py
import hmac
from fastapi import Header, HTTPException
from .config import settings
from . import warehouse

def current_client(x_gateway_secret: str = Header(...), x_apim_subscription: str = Header(...)) -> int:
    if not hmac.compare_digest(x_gateway_secret, settings.gateway_secret):
        raise HTTPException(status_code=401, detail="Not via gateway")
    rows = warehouse.query(
        "SELECT client_id FROM api.client_subscription WHERE apim_subscription_id = ?",
        x_apim_subscription)
    if not rows:
        raise HTTPException(status_code=403, detail="Unknown client")
    return rows[0]["client_id"]
```

In production, cache this lookup (for example with `functools.lru_cache` and a TTL) so it doesn't hit the Warehouse on every request.

### 5.6 Routes

```python
# app/main.py
from datetime import date
from fastapi import FastAPI, Depends, Query
from pydantic import BaseModel
from . import warehouse, semantic
from .security import current_client

app = FastAPI(title="ContosoData Client API", version="1.0.0")

class Sale(BaseModel):
    sale_id: int
    sale_date: date
    store: str | None
    product: str | None
    qty: int | None
    amount: float | None

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/v1/sales", response_model=list[Sale])
def list_sales(date_from: date, date_to: date,
               limit: int = Query(1000, le=10000), offset: int = 0,
               client_id: int = Depends(current_client)):
    return warehouse.query("""
        SELECT sale_id, sale_date, store, product, qty, amount
        FROM api.v_sales
        WHERE client_id = ? AND sale_date BETWEEN ? AND ?
        ORDER BY sale_date, sale_id
        OFFSET ? ROWS FETCH NEXT ? ROWS ONLY""",
        client_id, date_from, date_to, offset, limit)

@app.get("/v1/kpis/monthly")
def monthly_kpis(year: int, client_id: int = Depends(current_client)):
    # client filter is injected server-side; the caller never writes DAX
    dax = f"""
    EVALUATE
    SUMMARIZECOLUMNS(
        'Date'[Month],
        TREATAS({{ {int(client_id)} }}, 'Client'[client_id]),
        TREATAS({{ {int(year)} }}, 'Date'[Year]),
        "Sales", [Total Sales],
        "Units", [Total Units],
        "Avg Basket", [Average Basket]
    )"""
    return semantic.run_dax(dax)
```

Two things matter in this code. First, `client_id` always comes from `current_client` and never from the query string. Second, the DAX string only interpolates integers that have been cast with `int()`, so it can't be injected. Never interpolate free text into DAX or SQL.

### 5.7 Container and deployment

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y curl gnupg2 apt-transport-https \
 && curl -sSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/ms.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/ms.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" > /etc/apt/sources.list.d/mssql.list \
 && apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18 unixodbc-dev \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /srv
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# one-time
az group create -n rg-client-api -l westeurope
az acr create -n contosoapiacr -g rg-client-api --sku Basic
az containerapp env create -n cae-client-api -g rg-client-api -l westeurope

# build + deploy
az acr build -r contosoapiacr -t client-api:1.0.0 .
az containerapp create -n ca-client-api -g rg-client-api \
  --environment cae-client-api --image contosoapiacr.azurecr.io/client-api:1.0.0 \
  --registry-server contosoapiacr.azurecr.io --system-assigned \
  --target-port 8000 --ingress external --min-replicas 1 --max-replicas 5 \
  --secrets gateway-secret=<generated> \
  --env-vars SQL_SERVER=<xxxx>.datawarehouse.fabric.microsoft.com PBI_WORKSPACE_ID=<guid> \
             PBI_DATASET_ID=<guid> GATEWAY_SECRET=secretref:gateway-secret
```

After deployment, restrict Container Apps ingress to the APIM outbound IPs, or put both in a VNet with internal ingress.

### 5.8 API Management configuration

1. Create APIM (Developer tier for labs; Standard v2 or Premium for production).
2. **APIs → Add → OpenAPI:** import `https://<container-app-url>/openapi.json` and set the API URL suffix to `contoso`.
3. **Products:** create `Bronze` (1,000 calls/day) and `Gold` (50,000 calls/day). Each client gets a **subscription** to one product, which issues their key.
4. **Named value** `gateway-secret`, linked to Key Vault.
5. Inbound policy at API scope:

```xml
<policies>
  <inbound>
    <base />
    <rate-limit-by-key calls="20" renewal-period="1" counter-key="@(context.Subscription.Id)" />
    <quota-by-key calls="50000" renewal-period="86400" counter-key="@(context.Subscription.Id)" />
    <set-header name="X-Gateway-Secret" exists-action="override"><value>{{gateway-secret}}</value></set-header>
    <set-header name="X-Apim-Subscription" exists-action="override"><value>@(context.Subscription.Id)</value></set-header>
    <cache-lookup vary-by-developer="true" vary-by-developer-groups="false" downstream-caching-type="none">
      <vary-by-query-parameter>date_from;date_to;year;limit;offset</vary-by-query-parameter>
    </cache-lookup>
  </inbound>
  <outbound>
    <base />
    <cache-store duration="300" />
  </outbound>
</policies>
```

6. Register each client: insert a row into `api.client_subscription (apim_subscription_id, client_id)`.

### 5.9 What the client does

```bash
curl -H "Ocp-Apim-Subscription-Key: <their key>" \
  "https://contoso-apim.azure-api.net/contoso/v1/sales?date_from=2026-09-01&date_to=2026-09-30&limit=500"
```

The client never touches Entra ID or Fabric. They get a key, OpenAPI docs and a developer portal.

### 5.10 Lab check

- [ ] `/health` returns `ok` through APIM.
- [ ] Two subscriptions mapped to two clients get disjoint data.
- [ ] A call straight to the Container App without the gateway secret returns 401.
- [ ] The 21st call in one second returns HTTP 429.
- [ ] `/v1/kpis/monthly` matches the numbers in the Power BI report for that client.

## Module 6 — Securing the API

A client API is secure when five things hold: only known callers get in, each caller is identified individually, each caller sees only its own rows, secrets never live in code, and no single caller can exhaust your capacity.

### 6.1 Least-privilege identity for the API itself

The Container App's managed identity (or the per-client service principal, for GraphQL and UDF) needs exactly this and nothing more:

| Where | Grant | How |
| --- | --- | --- |
| Tenant settings | Allowed to call Fabric/Power BI APIs | Add the identity to `sg-fabric-api-clients`, which is the only group enabled for *Service principals can use Fabric APIs* |
| Warehouse item | **Read** (connect) — *not* ReadData, *not* a workspace role | Item → **Share** → untick all extra boxes |
| Warehouse SQL | `SELECT` on schema `api` only | `GRANT SELECT ON SCHEMA::api TO [ca-client-api];` |
| Semantic model | **Read** + **Build** (needed by executeQueries) | Model → **Manage permissions** |
| Workspace | No role, if you can avoid it | Item-level sharing instead |

Avoid giving the API identity **Contributor** or **Admin** on the workspace. With those roles it can read every item and bypass item-level SQL permissions.

### 6.2 Per-client isolation with Warehouse row-level security

The custom API filters by `client_id` in code. Defence in depth means the database enforces it too. For GraphQL and UDF with one identity per client, RLS is the *main* control.

```sql
-- 1. Security schema and mapping table
CREATE SCHEMA sec;
GO
CREATE TABLE sec.principal_client (principal_name VARCHAR(256) NOT NULL, client_id INT NOT NULL);
INSERT INTO sec.principal_client VALUES
 ('sp-client-northwind', 1),            -- service principal display name as seen by USER_NAME()
 ('analyst@fabrikam.com', 2);           -- B2B guest UPN
GO

-- 2. Predicate function
CREATE FUNCTION sec.fn_client_filter(@client_id INT)
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS allowed
       FROM sec.principal_client pc
       WHERE pc.client_id = @client_id
         AND pc.principal_name = USER_NAME()
       UNION ALL
       SELECT 1 WHERE IS_ROLEMEMBER('db_owner') = 1;  -- engineers see all
GO

-- 3. Policy on the base table: views over it inherit the filter
CREATE SECURITY POLICY sec.client_policy
ADD FILTER PREDICATE sec.fn_client_filter(client_id) ON core.sales
WITH (STATE = ON);
```

Test it with `EXECUTE AS USER = 'sp-client-northwind'; SELECT * FROM api.v_sales; REVERT;`. Check what `USER_NAME()` actually returns for a service principal in your tenant before relying on it; print it once from the API identity.

**For the custom API with one shared managed identity,** RLS by caller doesn't separate clients, because every client arrives as the same identity. Either keep the code filter and add an automated test that every route uses `current_client`, or set `SESSION_CONTEXT` per request and filter on it. Check that your Fabric Warehouse release supports `sp_set_session_context` before you choose this.

### 6.3 Column protection

- **Don't expose a column** in the `api` view. This is the strongest control.
- **Column-level security:** `GRANT SELECT ON core.sales(sale_id, sale_date, amount) TO [role_partner];`
- **Dynamic data masking:** `ALTER TABLE core.sales ALTER COLUMN customer_email ADD MASKED WITH (FUNCTION = 'email()');`. Users without `UNMASK` see `aXXX@XXXX.com`. Masking is cosmetic, not a security boundary, because ad-hoc queries can infer values.

### 6.4 Secrets

| Secret | Where it lives | Rotation |
| --- | --- | --- |
| API identity → Fabric | None: managed identity | n/a |
| APIM → API gateway secret | Key Vault, referenced by an APIM named value and a Container Apps secret | Every 90 days, scripted |
| Client keys | APIM subscriptions (primary + secondary) | Client rotates by switching to the secondary key |
| Client service principals (GraphQL/UDF) | Certificate preferred; secret max 6–12 months | Calendar reminder + Entra alert on expiry |

### 6.5 Abuse and cost controls

- **APIM** `rate-limit-by-key` (burst) and `quota-by-key` (daily/monthly), set per product.
- **Hard caps in code:** `limit ≤ 10000` and date range ≤ 366 days, so no single request can scan the whole table.
- **Caching:** APIM response cache for KPI endpoints. Pre-aggregated tables for heavy endpoints.
- **Isolate the capacity:** put the API's Warehouse on its own capacity (for example an F8) so a client spike can't throttle internal Power BI.
- **Surge protection** on the shared capacity (Module 10).

### 6.6 Threat checklist

| Threat | Control |
| --- | --- |
| Client A reads client B's data | Identity-derived `client_id` + RLS + automated cross-client test |
| SQL / DAX injection | Parameterised SQL; only cast integers interpolated into DAX |
| Stolen client key | APIM key regeneration; IP allow-list per product if the client has fixed IPs |
| Bypassing APIM | Gateway secret header + ingress IP restriction / private network |
| Bulk scraping | Quotas, row caps, monitoring of rows returned per client |
| Leaked engineer credentials | Conditional Access + MFA; PIM for admin roles (Module 11) |
| Data leaves via export | Sensitivity labels + DLP (Module 9) |

### 6.7 Exercise

Write a pytest that calls every route with client 1's subscription and asserts no row has `client_id != 1`, then run it in CI on every pull request.

## Module 7 — Governance foundations

Governance in Fabric starts with structure. Decide who owns which data (domains), where it lives (workspaces), how mature it is (endorsement) and how people find it (tags, descriptions, the OneLake catalog). Security controls come after that.

### 7.1 Governance model: centralised, federated or hybrid

| Model | Who builds | Who governs | Fabric mapping |
| --- | --- | --- | --- |
| Centralised | Central data team | Central team | Few workspaces, tenant settings tight |
| Federated (data mesh) | Each business domain | Domains, under central guardrails | Domains + domain admins + delegated settings |
| **Hybrid (recommended)** | Domains build Bronze/Silver; central team certifies Gold | Central CoE + domain stewards | Domains + certification restricted to the CoE |

### 7.2 Domains

- **Create** them in **Admin portal → Domains** (Fabric admin). Assign **domain admins** (who can delegate settings) and **domain contributors** (who can assign their workspaces to the domain).
- **Assign workspaces** individually, by workspace admin, or by capacity.
- **Subdomains** give finer grouping, for example `Sales → Retail`, `Sales → Wholesale`.
- **Delegated settings:** domain admins can override some tenant settings for their domain, such as certification reviewers and a default sensitivity label.
- In the OneLake catalog, users filter by domain to find data owned by that area.

For ContosoData, create domains `Client Products` (the API data), `Finance` and `Internal Analytics`.

### 7.3 Workspace design

Use workspaces to separate **stage**, **environment** and **audience**:

```
<domain>-<product>-<layer>-<env>
clientprod-sales-bronze-dev   clientprod-sales-bronze-prod
clientprod-sales-gold-dev     clientprod-sales-gold-prod     ← API reads only this one
clientprod-sales-reports-prod                                ← reports, apps for business users
```

| Workspace role | Can do | Give to |
| --- | --- | --- |
| **Admin** | Everything, including delete the workspace and manage access | 2 people max, through PIM |
| **Member** | Everything except delete or manage admins; can share | Lead engineers |
| **Contributor** | Create, edit and delete items; read all data | Engineers in dev; CI/CD identity in prod |
| **Viewer** | Read items; for SQL, only what's granted; for Lakehouse, through the SQL endpoint subject to SQL permissions | Analysts (better: item-level sharing or an app) |

Rules of thumb:

- Assign roles to **Entra groups**, never to individuals.
- **Prod workspaces:** no human Contributors. Changes arrive through deployment pipelines (Module 11).
- Business users consume through a **workspace app** or item sharing, not workspace roles.

### 7.4 Endorsement

| Badge | Who can apply | Meaning |
| --- | --- | --- |
| **Promoted** | Anyone with write access on the item | "I think this is ready to reuse" |
| **Certified** | Only users or groups the admin authorised (tenant or domain setting) | Reviewed against the CoE checklist |
| **Master data** | Only authorised users | Authoritative reference data (customers, products) |

An example certification checklist for an API-exposed Warehouse:

- [ ] Owner and steward named in the description
- [ ] Sensitivity label applied
- [ ] RLS policy in place and tested
- [ ] Data quality tests pass (row counts, nulls, freshness < 24 h)
- [ ] Source in Git, deployed through a pipeline
- [ ] Lineage reviewed: no Bronze dependencies

### 7.5 Tags and descriptions

- **Tags** are created by Fabric admins (centrally, or delegated to domains) and applied to items by item owners. Examples: `Client-API`, `Contains-PII`, `Gold`, `Deprecated`.
- Tags are searchable and filterable in the OneLake catalog and readable through the REST API, so they're good inputs for automation ("alert me if an item tagged `Client-API` loses its label").
- Write a **description** on every Gold item covering purpose, grain, refresh schedule and owner.

### 7.6 OneLake catalog

- **Explore** tab: everyone browses the items they can access, filtered by domain, type, endorsement and tag.
- **Govern** tab: data owners see insights and recommended actions about *their* items, such as items without labels, items not endorsed, or stale items with no recent refresh. Admins see a tenant-wide view.

### 7.7 Lab

1. Create the domain `Client Products` and assign `clientprod-sales-gold-dev` to it.
2. Create tags `Client-API` and `Contains-PII`, and tag `wh_gold`.
3. Allow only the group `sg-data-coe` to certify (tenant setting *Certification*).
4. Certify `wh_gold` after running through the checklist.
5. Open the OneLake catalog **Govern** tab and fix one recommendation.

## Module 8 — Data access security

Workspace roles decide who can open an item. Data-level controls decide which tables, columns and rows a person sees once inside. Fabric has three data-level layers — OneLake, the SQL engine and the semantic model — and you must know which one each engine honours.

### 8.1 Which layer applies to which engine

| Access path | Honours OneLake security | Honours SQL GRANT / RLS / CLS / DDM | Honours semantic-model RLS / OLS |
| --- | --- | --- | --- |
| Spark notebook on a Lakehouse | Yes | No | No |
| Lakehouse SQL analytics endpoint | Yes, in *user identity* mode | Yes | No |
| Warehouse T-SQL / GraphQL / UDF | n/a | Yes | No |
| Power BI report on the semantic model | Through the model's source | Only in DirectQuery or SSO | Yes |
| executeQueries / XMLA | Through the model's source | Same as above | Yes (unless the caller is a model admin or has write access) |

**Key trap:** workspace **Admin, Member and Contributor** roles bypass OneLake security and semantic-model RLS. Data-level security only restricts **Viewers** and people with item-level access.

### 8.2 OneLake security (Lakehouse)

OneLake security lets you define **roles** on a Lakehouse. A role says which tables or folders its members can read, optionally with row and column filters. It's enforced across engines: Spark, the SQL endpoint in user-identity mode and Direct Lake models.

Steps:

1. Lakehouse → **Manage OneLake security** → **New role** `role_partner_northwind`.
2. **Data:** select `Tables/sales`.
3. **Row security:** `SELECT * FROM sales WHERE client_id = 1`
4. **Column security:** hide `customer_email`.
5. **Members:** add the Entra group `sg-partner-northwind`.
6. Remove or narrow the **DefaultReader** role, which grants everyone with Read access all data.

OneLake security has gone through preview stages with changing limits, for example on which item types and shortcut types are supported. Verify the current status before using it for client isolation in production.

### 8.3 Warehouse SQL security

Granular permissions:

```sql
-- Roles instead of individual grants
CREATE ROLE role_api_reader;
GRANT SELECT ON SCHEMA::api TO role_api_reader;
ALTER ROLE role_api_reader ADD MEMBER [ca-client-api];          -- managed identity
ALTER ROLE role_api_reader ADD MEMBER [sg-partner-analysts];    -- Entra group

DENY SELECT ON SCHEMA::core TO role_api_reader;                 -- explicit deny
```

Column-level security:

```sql
GRANT SELECT ON core.sales (sale_id, client_id, sale_date, store, product, qty, amount) TO role_partner;
-- SELECT * now fails for role_partner; they must name the allowed columns
```

Dynamic data masking:

```sql
ALTER TABLE core.sales ALTER COLUMN customer_email ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE core.sales ALTER COLUMN amount ADD MASKED WITH (FUNCTION = 'default()');
GRANT UNMASK TO [sg-finance];
```

Row-level security: see the full example in Module 6.2 (predicate function + security policy).

**Direct Lake interaction:** if a semantic model in Direct Lake mode reads a Warehouse or SQL endpoint table protected by SQL RLS, CLS or OLS, queries **fall back to DirectQuery**, which is slower. Model-side RLS doesn't cause fallback. Either secure in the model, or accept the fallback.

### 8.4 Semantic-model RLS and OLS

Dynamic RLS from a mapping table:

```dax
// Model: table 'SecurityMap' [upn], [client_id], related to 'Client'[client_id]
// Role "ClientUser", filter on 'SecurityMap':
'SecurityMap'[upn] = USERPRINCIPALNAME()
```

If `SecurityMap` filters `Client`, and `Client` filters `Sales` (single direction, with *Apply security filter in both directions* enabled where needed), each user sees only their client.

Static role example:

```dax
// Role "Northwind" on table 'Client'
'Client'[client_id] = 1
```

**Object-level security (OLS)** hides whole tables or columns. Configure it with Tabular Editor or TMDL by setting the role's `metadataPermission` on the column to `none`:

```
role ClientUser
    modelPermission: read
    tablePermission Sales
        columnPermission customer_email = none
```

**Test** with **Security → View as role** in Power BI Desktop or the service, or from DAX Studio with `EffectiveUserName` / `Roles` in the connection string.

### 8.5 Item permissions cheat sheet

| Permission | Grants |
| --- | --- |
| Read | Connect to the item (see metadata) |
| ReadData | Read all data through the SQL endpoint, subject to SQL permissions |
| ReadAll | Read the underlying OneLake files (Spark, OneLake APIs) |
| Build | Build new content on a semantic model; required for executeQueries |
| Reshare | Share the item onwards |
| Write | Edit the item; also bypasses model RLS |
| Execute | Run a GraphQL API or User Data Function |

### 8.6 Lab

1. Create `role_api_reader` and confirm a test user can query `api.v_sales` but not `core.sales`.
2. Add masking to `customer_email` and query as a non-UNMASK user.
3. Build dynamic RLS on the `Sales Analytics` model and test it with *View as*.
4. Check whether the Direct Lake model falls back to DirectQuery after you add SQL RLS, using Performance Analyzer or the `DirectLakeBehavior` setting.

## Module 9 — Microsoft Purview with Fabric

Purview adds the controls that follow data *beyond* Fabric's access model: classification labels that travel with exports, policies that detect sensitive data, an enterprise catalog, and an audit trail. Most of it works on Fabric data with no extra connector, but some features need Microsoft 365 E3/E5 or Purview licences.

### 9.1 Sensitivity labels (Purview Information Protection)

**What they do in Fabric:**

- A label is set on any item (Lakehouse, Warehouse, model, report, notebook…) and shown in the header and the catalog.
- **Downstream inheritance:** a label on a Lakehouse flows to the models and reports built on it, so they're never less sensitive than their source.
- **Inheritance from data sources:** labels can be inherited when data is imported from labelled sources, such as Excel or an Azure SQL database scanned by Purview.
- **Export protection:** when data is exported to Excel, PowerPoint or PDF, the file carries the label. If the label has encryption, only authorised people can open the file — even outside the company.
- **Policy options** (set in Purview, enabled in Fabric tenant settings): mandatory labelling, default label, and justification required to lower a label.

**Setup:**

1. In the Purview portal, under **Information Protection → Labels**, create a taxonomy, for example `Public / General / Confidential / Highly Confidential`, with encryption on the top two.
2. Publish a **label policy** to the Fabric users.
3. In **Fabric Admin portal → Information protection**, enable *Allow users to apply sensitivity labels*, *Apply sensitivity labels from data sources*, *Automatically apply labels to downstream content*, and optionally *Mandatory label* and *Default label* (also available per domain).
4. Label `wh_gold` as **Confidential** and confirm the `Sales Analytics` model inherits it.

### 9.2 Data loss prevention (DLP) for Fabric

DLP policies scan Fabric items (semantic models and supported data items such as Lakehouses) for **sensitive information types**, such as credit card numbers, IBANs or national IDs, and for sensitivity labels.

When a match is found, DLP can:

- show a **policy tip** on the item,
- send an alert to the compliance team and email the owner,
- **restrict access** for some item types (for example, limit to owner/admins until fixed).

Example policy: *"Any Fabric item in workspace `clientprod-*` containing EU IBAN or email addresses → policy tip + alert to dpo@contoso.com; if label is lower than Confidential → restrict access."*

Create it in **Purview → Data loss prevention → Policies → Create policy → Custom**. Choose **Fabric and Power BI** as the location, then scope to workspaces.

### 9.3 Purview Unified Catalog and Data Map

- **Register** your Fabric tenant as a source in the Purview Data Map and run a **scan**. It pulls workspaces, items, schemas and lineage into the enterprise catalog, next to your SQL Server, Databricks, Snowflake or SAP sources.
- **Unified Catalog** adds business-level governance:
  - **Governance domains** (business ownership; distinct from Fabric domains, but often mirrored),
  - **Data products** (for example "Client Sales API dataset", backed by `wh_gold.api.v_sales`), with owners, terms of use and an access-request workflow,
  - **Glossary terms** ("Net Sales", "Active Client"),
  - **Data quality** rules and scores on supported sources,
  - **Health controls** (for example, % of data products with owners).
- Use this when your estate spans more than Fabric. For Fabric-only shops, the OneLake catalog may be enough.

### 9.4 Audit

Every Fabric operation — view report, export, share, create item, change tenant setting, run query through the API — is written to the **Microsoft 365 unified audit log**.

- Search interactively in **Purview → Audit** (filter on workload *PowerBI* / *MicrosoftFabric*).
- Pull programmatically through the **Power BI Admin Activity Events API**, which is good for daily ingestion into a Lakehouse:

```python
import requests, datetime as dt
from azure.identity import ClientSecretCredential

token = ClientSecretCredential(TENANT, ADMIN_SP_ID, SECRET).get_token(
    "https://analysis.windows.net/powerbi/api/.default").token
day = (dt.date.today() - dt.timedelta(days=1)).isoformat()
url = ("https://api.powerbi.com/v1.0/myorg/admin/activityevents"
       f"?startDateTime='{day}T00:00:00Z'&endDateTime='{day}T23:59:59Z'")
events = []
while url:
    r = requests.get(url, headers={"Authorization": f"Bearer {token}"}).json()
    events += r.get("activityEventEntities", [])
    url = None if r.get("lastResultSet") else r.get("continuationUri")
print(len(events), "events for", day)
```

- Retention is 180 days by default (longer with premium audit licences), so archive to a Lakehouse if you need more.
- Useful queries: exports per user per day, shares to external users, label downgrades, tenant-setting changes.

### 9.5 Other Purview capabilities that touch Fabric

| Capability | Use |
| --- | --- |
| **Purview hub** (inside Fabric) | Label and endorsement coverage reports for admins and owners |
| **Data Security Posture Management (DSPM), incl. for AI** | Find oversharing and risky Copilot/AI use over sensitive data |
| **Insider Risk Management** | Detect, for example, a leaving employee mass-exporting reports |
| **eDiscovery / retention** | Legal holds over content where supported |

### 9.6 Lab

1. Publish the label taxonomy and label `wh_gold` Confidential. Check that inheritance reaches the report.
2. Export a report to Excel and open it from a personal account. It should be blocked if the label encrypts.
3. Create a DLP policy for email addresses in `clientprod-*` and trigger it with the `customer_email` column.
4. Ingest yesterday's audit events into a Lakehouse table `gov.audit_events`.

## Module 10 — Admin, monitoring and capacity governance

Tenant settings define what is *possible*. Monitoring tells you what is *happening*. Capacity controls keep one workload from hurting the others.

### 10.1 Tenant settings that matter most

Scope every sensitive setting to a **security group**, never "the entire organization".

| Setting (Admin portal → Tenant settings) | Recommended |
| --- | --- |
| Users can create Fabric items | Enabled for `sg-fabric-creators` |
| Service principals can use Fabric APIs | Enabled for `sg-fabric-api-clients` |
| Service principals can access read-only admin APIs | Enabled for `sg-governance-automation` |
| Dataset Execute Queries REST API | Enabled (needed by the custom API) |
| Allow XMLA endpoints and Analyze in Excel | Enabled for BI developers |
| Publish to web | **Disabled** |
| Export to Excel / CSV / PDF | Enabled for internal staff; disabled for B2B guests |
| Guest users can access Fabric | Enabled for `sg-client-guests` only |
| Users can share with external users | Disabled, or restricted to `sg-partner-managers` |
| Certification | Enabled for `sg-data-coe` |
| Workspace creation | Restricted to `sg-workspace-creators` |
| Copilot and Azure OpenAI | Enabled for a pilot group; decide on cross-geo processing |
| Allow workspace admins to delegate | Decide per setting (domain/capacity/workspace delegation) |

### 10.2 Admin monitoring workspace

In **Admin portal → Admin monitoring workspace** (auto-created, admin-only) you find two things:

- **Feature Usage and Adoption** report: active users, items, operations, top workspaces.
- **Purview Hub (admin)**: label and endorsement coverage.

You can share these with a governance group. Their semantic models can be reused for custom reports.

### 10.3 Monitoring hub and workspace monitoring

- **Monitoring hub** (left nav): run history for pipelines, notebooks, dataflows and refreshes, with filters by status and duration. Use it for operations.
- **Workspace monitoring:** turn it on in workspace settings. Fabric creates an **Eventhouse** in the workspace that collects detailed logs (semantic-model queries, GraphQL requests, Eventhouse queries and more) for about 30 days. You query them with KQL:

```kusto
// Slowest semantic-model queries in the last 24 h
SemanticModelLogs
| where Timestamp > ago(1d) and OperationName == "QueryEnd"
| top 20 by DurationMs desc
| project Timestamp, ExecutingUser, DurationMs, EventText
```

```kusto
// GraphQL API calls by caller
GraphQLMetrics
| where Timestamp > ago(1d)
| summarize calls = count(), p95_ms = percentile(OverallDurationMs, 95) by CallerId = tostring(column_ifexists("UserId", ""))
```

Table and column names in workspace monitoring evolve. Open the Eventhouse and inspect the schema before you build alerts on it.

### 10.4 Capacity governance

- **Fabric Capacity Metrics app:** install it from AppSource and connect it to your capacity ID. It shows CU % by item and operation, **throttling** (interactive delay, interactive rejection, background rejection) and **smoothing**.
- **How throttling works:** usage is smoothed over time (interactive over minutes, background over 24 h). When future capacity is overconsumed, Fabric first delays and then rejects new requests.
- **Surge protection:** set a background-rejection threshold, so runaway background jobs (Spark, pipelines) are rejected before they starve interactive reports and APIs.
- **Separate capacities** by criticality: for example `cap-client-api` (F8, API Warehouse only), `cap-internal` (F64) and `cap-dev` (F4, pausable).
- **Pause and resume** dev capacities out of hours through Azure automation to save cost.
- **Spark autoscale billing** (pay-as-you-go Spark outside the capacity) where available, to keep big Spark jobs off shared CU.

### 10.5 Admin REST APIs and the Scanner API

The Scanner API is the backbone of custom governance reporting. It returns every workspace with its items, users, endorsement, labels, data sources, lineage and (optionally) table schemas and DAX.

```python
import requests, time
H = {"Authorization": f"Bearer {pbi_admin_token}"}
BASE = "https://api.powerbi.com/v1.0/myorg/admin/workspaces"

# 1. Workspaces modified since a date (or all)
ws = requests.get(f"{BASE}/modified?excludePersonalWorkspaces=true", headers=H).json()
ids = [w["id"] for w in ws][:100]                     # max 100 per scan request

# 2. Start scan
scan = requests.post(f"{BASE}/getInfo?lineage=true&datasourceDetails=true&datasetSchema=true&datasetExpressions=true&getArtifactUsers=true",
                     headers=H, json={"workspaces": ids}).json()

# 3. Poll then fetch
while requests.get(f"{BASE}/scanStatus/{scan['id']}", headers=H).json()["status"] != "Succeeded":
    time.sleep(5)
result = requests.get(f"{BASE}/scanResult/{scan['id']}", headers=H).json()
```

Land `result` in a Lakehouse daily and build a **governance dashboard** from it: items without labels, Gold items not certified, workspaces with individual users instead of groups, external users with access, and service principals with Admin roles.

The Fabric REST API also offers admin endpoints (`api.fabric.microsoft.com/v1/admin/...`) for items, domains, tags, tenant settings and access. Use them to automate domain assignment and tagging.

### 10.6 Lab

1. Export the current tenant settings with `GET https://api.fabric.microsoft.com/v1/admin/tenantsettings` and store them in Git as a baseline.
2. Turn on workspace monitoring for `clientprod-sales-gold-prod` and write a KQL query for API calls per client per hour.
3. Install the Capacity Metrics app and identify the top three CU consumers.
4. Run the Scanner API and list every item without a sensitivity label.

## Module 11 — ALM and network governance

Change control makes sure nothing reaches production without review. Network and identity controls make sure only trusted paths reach the data.

### 11.1 Git integration

- Connect a workspace to **Azure DevOps** or **GitHub** (workspace settings → **Git integration**), on a branch and folder.
- Supported items (Lakehouse metadata, Warehouse, notebooks, pipelines, semantic models in **TMDL**, reports in **PBIR**, GraphQL APIs, UDFs and more) are serialised as text files. Data isn't in Git.
- Pattern: each engineer works in a **feature workspace** branched from `main`, opens a **pull request**, and a reviewer approves it. The `dev` workspace syncs from `main`.
- Branch policies in Git (required reviewers, build validation) are your real approval gate.

Example build-validation step, which fails the PR if a model has no RLS role:

```bash
# in Azure DevOps / GitHub Actions
grep -rq "^role " SalesAnalytics.SemanticModel/definition/roles/ || { echo "No RLS role defined"; exit 1; }
```

### 11.2 Deployment pipelines

- Up to 10 stages, typically **Dev → Test → Prod**, each bound to a workspace.
- **Deploy** compares stages and promotes the selected items.
- **Deployment rules / variable libraries** swap environment values, such as data source connections, Lakehouse IDs and parameters, so prod points at prod data.
- Restrict who can deploy to Prod with pipeline roles plus the Prod workspace role.
- Automate through the **Fabric REST API** (`deploymentPipelines/{id}/deploy`) or the `fabric-cicd` Python library from a CI/CD pipeline with a service principal. Humans then never need Contributor on Prod.

```python
# fabric-cicd: publish a Git folder into a target workspace
from fabric_cicd import FabricWorkspace, publish_all_items
ws = FabricWorkspace(workspace_id="<prod-ws-guid>", repository_directory="./workspace",
                     item_type_in_scope=["Notebook", "DataPipeline", "SemanticModel", "Report", "Warehouse"],
                     environment="PROD")
publish_all_items(ws)
```

### 11.3 Network controls

| Control | Scope | What it does | When to use |
| --- | --- | --- | --- |
| **Private Link (tenant level)** | Whole tenant | Fabric reachable only through private endpoints in your VNet; public access can be blocked | Highly regulated tenants. Affects many features, so test first |
| **Private Link (workspace level)** | One workspace | Inbound access to that workspace only through its private endpoint | Isolate the API's Gold workspace |
| **Workspace IP firewall rules** | One workspace | Allow inbound only from listed public IP ranges | Lighter alternative to Private Link |
| **Managed virtual network** | Spark in a workspace | Spark runs in an isolated VNet | Required for managed private endpoints |
| **Managed private endpoints** | Workspace → Azure source | Spark and some items reach Azure SQL, Storage and Key Vault privately | Sources with public access disabled |
| **Trusted workspace access** | Workspace identity → ADLS Gen2 | Shortcuts and pipelines reach firewalled storage accounts | Storage behind a firewall |
| **Outbound access protection** | Workspace | Blocks Spark/items from calling arbitrary internet endpoints (exfiltration) | Sensitive workspaces |
| **On-premises / VNet data gateway** | Connections | Reach private sources without opening inbound ports | Pipelines, dataflows, models |

### 11.4 Identity controls

- **Conditional Access** on the *Microsoft Fabric / Power BI Service* cloud app: require MFA, a compliant device, and named locations for admins. Apply separate policies for B2B guests.
- **Privileged Identity Management (PIM):** make Fabric Administrator and Prod workspace Admin *eligible* roles, activated just in time with approval and a time limit. For workspace roles, assign them to a PIM-for-Groups group.
- **Workspace identity:** a managed identity tied to the workspace. Use it for trusted workspace access and connections instead of personal credentials.
- **Service principal hygiene:** certificates over secrets, an owner on every app, quarterly access reviews (Entra **Access Reviews**) of groups that grant Fabric access.

### 11.5 Data protection at rest and residency

- Data is encrypted at rest with Microsoft-managed keys by default.
- **Customer-managed keys (CMK)** at workspace level use a key in your Azure Key Vault. Revoking the key makes the data unreadable. Check which item types are supported in your region.
- **Multi-geo:** create capacities in specific regions so a workspace's data at rest stays there, for example EU client data in West Europe. Note that some metadata stays in the home region.

### 11.6 Lab

1. Connect `clientprod-sales-gold-dev` to a GitHub repository and commit the Warehouse and model.
2. Create a deployment pipeline Dev → Prod, with a deployment rule for the model's Warehouse connection.
3. Write a GitHub Action that runs `fabric-cicd` with a service principal on merge to `main`.
4. Add a workspace IP firewall rule on Prod that allows only the APIM/Container Apps egress IPs and your office range, then confirm access from elsewhere is denied.

## Module 12 — Capstone, quiz and cheat sheet

### 12.1 Capstone project

**Brief:** onboard a new client, *Tailspin*, to the ContosoData API in under one day, with full governance evidence.

Deliverables:

- [ ] `api.v_sales` and `api.v_kpi_monthly` views, in Git, deployed to Prod by pipeline
- [ ] Tailspin mapped in `api.client_subscription` and `sec.principal_client`
- [ ] APIM subscription on the `Bronze` product; key delivered through a secure channel
- [ ] Automated cross-client test passing in CI
- [ ] `wh_gold` labelled Confidential, certified, tagged `Client-API`, in domain `Client Products`
- [ ] DLP policy covering `clientprod-*`
- [ ] Workspace-monitoring KQL showing Tailspin's first 100 calls
- [ ] A one-page **data-sharing record**: data shared, legal basis, retention, owner, review date

**Assessment:** a reviewer tries to read another client's data with Tailspin's key, bypass APIM, and export a report without a label. All three must fail.

### 12.2 Knowledge check

1. Which Fabric API option can't query a semantic model?
2. Which tenant setting must be enabled before any service principal can call Fabric APIs?
3. Why should a client API read from views in an `api` schema rather than tables?
4. A Contributor on the workspace opens a report on a model with RLS. What do they see?
5. What happens to a Direct Lake model when the Warehouse table it reads has SQL RLS?
6. Which two item permissions does a caller need for executeQueries?
7. Where is an APIM subscription key validated, and how does the backend know which client called?
8. What's the difference between Promoted and Certified?
9. Which tool lets you list every item without a sensitivity label across the tenant, programmatically?
10. Name two ways to stop a runaway client from throttling internal reports.

**Answers**

1. API for GraphQL.
2. *Service principals can use Fabric APIs* (scoped to a security group).
3. A stable contract: you can refactor tables, hide columns and add filters without breaking clients.
4. All data. Admin, Member and Contributor roles bypass semantic-model RLS.
5. Queries fall back to DirectQuery.
6. Read and Build on the semantic model (plus the *Dataset Execute Queries REST API* tenant setting).
7. In APIM. The policy forwards the subscription ID (and a gateway secret), and the backend maps it to a `client_id`.
8. Anyone with write access can Promote. Only admin-authorised reviewers can Certify.
9. The Scanner API (Admin `getInfo` / `scanResult`), or the Purview hub report.
10. Any two of: APIM rate limits and quotas; a separate capacity for the API; surge protection; row and date-range caps; caching.

### 12.3 Cheat sheet

| Need | Use |
| --- | --- |
| Quick API on Warehouse, no code | API for GraphQL (SSO mode for per-caller RLS) |
| Custom logic or write-back inside Fabric | User Data Functions |
| Paying external clients, keys, quotas | FastAPI on Container Apps + APIM |
| Governed KPIs from the model | executeQueries (small results) or XMLA (heavy) |
| Group workspaces by business | Domains |
| Signal trust | Endorsement: Promoted / Certified / Master data |
| Hide rows | Warehouse RLS, OneLake security, model RLS |
| Hide columns | Don't expose → CLS → OLS → masking (weakest) |
| Classify and protect exports | Sensitivity labels |
| Detect sensitive data | Purview DLP |
| Who did what | Purview Audit / Activity Events API |
| Inventory and lineage at scale | Scanner API, lineage view, Purview Data Map |
| CU usage and throttling | Capacity Metrics app, surge protection |
| Safe change | Git + deployment pipelines + fabric-cicd |
| Private access | Private Link, IP firewall, managed private endpoints |
| Just-in-time admin | Entra PIM + Conditional Access |

### Further reading

Check every feature against current Microsoft Learn pages before you deliver this course. Fabric changes monthly, and several features named here were in preview at various points. The links below follow Microsoft Learn's usual paths but weren't individually checked for this draft.

- [Microsoft Fabric API for GraphQL](https://learn.microsoft.com/fabric/data-engineering/api-graphql-overview)
- [Fabric User Data Functions](https://learn.microsoft.com/fabric/data-engineering/user-data-functions/user-data-functions-overview)
- [Datasets – Execute Queries REST API](https://learn.microsoft.com/rest/api/power-bi/datasets/execute-queries)
- [Security in Microsoft Fabric](https://learn.microsoft.com/fabric/security/security-overview)
- [Governance and compliance in Fabric](https://learn.microsoft.com/fabric/governance/governance-compliance-overview)
- [OneLake security](https://learn.microsoft.com/fabric/onelake/security/get-started-security)
- [Admin monitoring workspace](https://learn.microsoft.com/fabric/admin/monitoring-workspace)
- [Metadata scanning (Scanner API)](https://learn.microsoft.com/fabric/governance/metadata-scanning-overview)
- [Deployment pipelines](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/intro-to-deployment-pipelines)
- [Azure API Management policies](https://learn.microsoft.com/azure/api-management/api-management-policies)
