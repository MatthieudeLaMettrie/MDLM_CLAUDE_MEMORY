# Snowflake — Data APIs & Governance: Full Course

> Source of truth: the live Claude Doc (https://claude.ai/code/artifact/26c8e947-0a29-4df1-a791-47b58417e15c), which also holds the drawn diagrams. Exported 2026-10-13. Written from model knowledge; preview-era syntax is flagged in the text for verification.


Oct 13, 2026 · @Matt

## Course overview

By the end of this course you can serve Snowflake data and governed metrics to external clients through the SQL API, Snowpark Container Services, Secure Data Sharing or your own gateway-fronted API, and govern the account with Snowflake Horizon: RBAC, policies, tags, access history and cost controls.

**Audience:** data engineers, platform owners and analytics leads who know SQL and some Python.

**Prerequisites:** a Snowflake account (Enterprise edition or higher for masking, row access and tag-based policies; Business Critical for PrivateLink and Tri-Secret Secure), `ACCOUNTADMIN` for setup, Python 3.10+, and Snowflake CLI (`snow`).

**Running scenario.** *ContosoData* sells retail sales analytics. Each client must get only its own rows and the same KPI definitions you use internally. Data lives in database `CONTOSO_PROD`.

### Learning outcomes

1. Explain organization, account, warehouse, role and object hierarchy, and where each control lives.
2. Choose between the SQL API, SPCS, Secure Data Sharing and a custom API for a client need.
3. Build each option with key-pair, OAuth or workload-identity authentication.
4. Isolate clients with row access policies and protect columns with masking policies.
5. Govern with role design, tags, classification, access history, data metric functions and budgets.
6. Ship changes through Git and CI/CD behind authentication and network policies.

### Module map

| # | Module | Type | Time |
| --- | --- | --- | --- |
| 1 | Architecture essentials | Concepts | 45 min |
| 2 | API options compared | Concepts | 30 min |
| 3 | SQL API | Lab | 60 min |
| 4 | FastAPI on SPCS | Lab | 2 h |
| 5 | Secure Data Sharing | Lab | 60 min |
| 6 | External FastAPI + semantic views | Lab | 2 h |
| 7 | Securing the API | Concepts + lab | 60 min |
| 8 | Governance foundations | Concepts + lab | 60 min |
| 9 | Fine-grained policies | Lab | 90 min |
| 10 | Audit, monitoring, cost | Lab | 60 min |
| 11 | ALM, identity, network | Concepts + lab | 60 min |
| 12 | Capstone + quiz | Project | 3 h |

### Shared setup script

```sql
USE ROLE SYSADMIN;
CREATE WAREHOUSE IF NOT EXISTS WH_API WAREHOUSE_SIZE = XSMALL AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE COMMENT = 'Client API workloads';
CREATE DATABASE IF NOT EXISTS CONTOSO_PROD;
CREATE SCHEMA IF NOT EXISTS CONTOSO_PROD.GOLD;
CREATE SCHEMA IF NOT EXISTS CONTOSO_PROD.API WITH MANAGED ACCESS;
CREATE SCHEMA IF NOT EXISTS CONTOSO_PROD.SEC WITH MANAGED ACCESS;

CREATE OR REPLACE TABLE CONTOSO_PROD.GOLD.SALES (
  SALE_ID NUMBER, CLIENT_ID NUMBER, SALE_DATE DATE, STORE STRING,
  PRODUCT STRING, QTY NUMBER, AMOUNT NUMBER(18,2), CUSTOMER_EMAIL STRING);
INSERT INTO CONTOSO_PROD.GOLD.SALES VALUES
 (1,1,'2026-09-01','Paris-01','Coffee',3,12.60,'a@x.com'),
 (2,1,'2026-09-02','Paris-02','Tea',1,4.20,'b@x.com'),
 (3,2,'2026-09-01','Lyon-01','Coffee',5,21.00,'c@y.com');

CREATE OR REPLACE SECURE VIEW CONTOSO_PROD.API.V_SALES AS
SELECT SALE_ID, CLIENT_ID, SALE_DATE, STORE, PRODUCT, QTY, AMOUNT
FROM CONTOSO_PROD.GOLD.SALES;
```

Snowflake releases weekly. Features such as semantic views, Cortex Analyst, programmatic access tokens and workload identity federation were in preview at various points, so confirm their status in your region.

## Module 1 — Snowflake architecture essentials

Snowflake separates storage, compute (virtual warehouses) and cloud services (metadata, security, optimisation). Every access decision is made by the **role** active in the session, and every object belongs to exactly one role.

### 1.1 The hierarchy

| Level | What it is | Admin | Key controls |
| --- | --- | --- | --- |
| **Organization** | All your accounts across clouds and regions | `ORGADMIN` / organization account | Account creation, replication groups, org-wide usage |
| **Account** | One Snowflake deployment in one cloud region | `ACCOUNTADMIN` | Users, roles, network and authentication policies, parameters, shares |
| **Warehouse** | Compute cluster(s) that run queries | Owner role | Size, auto-suspend, multi-cluster, resource monitor |
| **Database → Schema** | Containers for objects | Owner role | Grants, managed access, database roles, replication |
| **Object** | Table, view, secure view, semantic view, stage, function, procedure, stream, task | Owner role | Grants, masking / row access / projection / aggregation policies, tags |

Objects are addressed as `DATABASE.SCHEMA.OBJECT`, for example `CONTOSO_PROD.GOLD.SALES`.

### 1.2 Roles: RBAC with ownership

- **Discretionary + role-based:** privileges are granted to roles, roles are granted to users or to other roles, and the object's **owner** role controls grants (unless the schema uses *managed access*).
- **System roles:** `ORGADMIN` → `ACCOUNTADMIN` → (`SECURITYADMIN` → `USERADMIN`) and `SYSADMIN`, plus `PUBLIC`. Custom roles should roll up to `SYSADMIN` so admins can manage the objects they create.
- **Database roles** are scoped to one database. They're ideal for shares and for packaging access (Module 8).
- A session has one **primary role** plus optional **secondary roles** (`USE SECONDARY ROLES ALL`).

### 1.3 User types and authentication

| User `TYPE` | Use for | Allowed authentication |
| --- | --- | --- |
| `PERSON` | Humans | SSO (SAML / OAuth), password + MFA (being phased out for passwords) |
| `SERVICE` | APIs, pipelines, client integrations | **Key-pair (JWT)**, OAuth client credentials, **workload identity federation**, programmatic access tokens. No password, no MFA prompts |
| `LEGACY_SERVICE` | Old integrations still on passwords | Transitional only; migrate away |

Authentication methods for machines:

- **Key-pair:** you register an RSA public key on the user, and the client signs a short-lived JWT with its private key. It works everywhere, including the SQL API.
- **OAuth:** Snowflake OAuth (security integration) or **External OAuth** (Entra ID, Okta, Ping), where the token's role scope maps to a Snowflake role.
- **Programmatic access tokens (PATs):** a token scoped to one user and optionally one role, with an expiry. They need a network policy on the user.
- **Workload identity federation:** an AWS IAM role, Azure managed identity, GCP service account or OIDC issuer (GitHub Actions) authenticates directly, with no secret stored.

Account identifier format: `<orgname>-<account_name>`, for example `contoso-prod`, giving host `contoso-prod.snowflakecomputing.com`.

### 1.4 Warehouses for API workloads

- Use a **dedicated warehouse** (`WH_API`), so client load is isolated and billable separately.
- `AUTO_SUSPEND = 60` seconds and `AUTO_RESUME = TRUE` keep cost proportional to use.
- **Multi-cluster** (Enterprise+) scales out for concurrency: `MIN_CLUSTER_COUNT = 1, MAX_CLUSTER_COUNT = 3, SCALING_POLICY = STANDARD`.
- `STATEMENT_TIMEOUT_IN_SECONDS = 120` on the warehouse or the API user stops runaway queries.
- The **result cache** (24 h, per identical query and role context) returns repeat queries without warehouse cost.

### 1.5 Exercise

1. Run the setup script.
2. Create a service user with key-pair authentication:

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

```sql
USE ROLE USERADMIN;
CREATE USER SVC_CLIENT_NORTHWIND TYPE = SERVICE
  DEFAULT_WAREHOUSE = WH_API
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqh...';   -- body of rsa_key.pub without header/footer
DESC USER SVC_CLIENT_NORTHWIND;          -- note RSA_PUBLIC_KEY_FP
```

## Module 2 — API options compared

Snowflake's strongest native answer to "clients need our data" is **Secure Data Sharing**: zero-copy, live, and revocable, with **reader accounts** for clients who don't run Snowflake. For request/response APIs you can use the **SQL API**, host FastAPI in **Snowpark Container Services**, or run your own service behind a gateway.

### 2.1 Side-by-side comparison

| Criterion | SQL API | Snowpark Container Services | Secure Data Sharing / listings | External FastAPI + gateway | Cortex Analyst REST API |
| --- | --- | --- | --- | --- | --- |
| Hosted in | Snowflake | Snowflake (containers) | Snowflake | Your cloud | Snowflake |
| Protocol | REST, SQL in JSON | REST you design | Client queries in its own (or reader) account | REST you design | REST, natural language → SQL |
| Client auth | Key-pair JWT, OAuth, PAT | Snowflake OAuth / PAT on the public endpoint | Snowflake account-to-account | API keys, OAuth — your choice | Same as SQL API |
| Quotas, keys, portal | No | Build it yourself | Per-consumer access | Yes (APIM, AWS API Gateway, Apigee) | No |
| Compute paid by | You | You (compute pool) | **Consumer** pays queries (you pay for reader accounts) | You | You |
| Best for | Simple integrations, internal apps | Custom APIs next to the data | Clients wanting live tables | Commercial API products | Conversational client portals |

### 2.2 What each option is

**SQL API** (`POST /api/v2/statements`): submit one statement (or a multi-statement batch) with bind variables, get a JSON result set in partitions. It runs as the authenticated user with its default or specified role and warehouse.

**Snowpark Container Services (SPCS)** runs OCI containers on a **compute pool** inside Snowflake. A *service* can expose a **public endpoint** (HTTPS ingress, authenticated by Snowflake), and code inside reaches Snowflake data through a token injected into the container. No credentials are needed.

**Secure Data Sharing** grants another Snowflake account read access to objects in your account, with no copy and no ETL. **Listings** (private or on the Marketplace) package a share with metadata, terms and auto-fulfilment across regions. **Reader accounts** are Snowflake accounts you create and pay for, for consumers without Snowflake.

**External FastAPI + gateway** uses `snowflake-connector-python` as a service user. It's the pattern for API products with keys and quotas.

**Semantic views** define tables, relationships, dimensions, facts and metrics once in Snowflake. They're queried with SQL or through **Cortex Analyst**, which turns natural-language questions into SQL against the semantic view.

**Hybrid tables** (Unistore) and **Snowflake Postgres** serve low-latency point lookups where standard tables are too slow for high-QPS APIs.

### 2.3 Decision tree

&#91;embedded content: Snowflake API option decision tree · 4 questions, 5 outcomes\]

Stop at the first "yes". Sharing comes first because it costs you the least to run: the consumer pays for its own queries and you maintain no endpoint.

### 2.4 Common mistakes

- **Using a person's password** for an integration. Use `TYPE = SERVICE` users with key-pair or federation.
- **Sharing a regular view.** Only **secure** views (and secure UDFs) can be shared, and they hide the view definition.
- **One warehouse for everything.** Client queries then compete with ELT and BI. Use a dedicated `WH_API`.
- **Forgetting the role.** The SQL API runs with the user's default role unless you pass `"role"`, so set a least-privileged default role.

## Module 3 — Lab: Snowflake SQL API

The SQL API lets any HTTP client run SQL as a Snowflake user. In this lab the client's service user `SVC_CLIENT_NORTHWIND` authenticates with a key-pair JWT and pulls its sales.

### 3.1 Least-privilege role

```sql
USE ROLE SECURITYADMIN;
CREATE ROLE IF NOT EXISTS R_CLIENT_NORTHWIND;
GRANT USAGE ON WAREHOUSE WH_API TO ROLE R_CLIENT_NORTHWIND;
GRANT USAGE ON DATABASE CONTOSO_PROD TO ROLE R_CLIENT_NORTHWIND;
GRANT USAGE ON SCHEMA CONTOSO_PROD.API TO ROLE R_CLIENT_NORTHWIND;
GRANT SELECT ON VIEW CONTOSO_PROD.API.V_SALES TO ROLE R_CLIENT_NORTHWIND;
GRANT ROLE R_CLIENT_NORTHWIND TO USER SVC_CLIENT_NORTHWIND;
ALTER USER SVC_CLIENT_NORTHWIND SET DEFAULT_ROLE = R_CLIENT_NORTHWIND;
```

The row access policy in Module 9 makes `V_SALES` return only client 1's rows for this role.

### 3.2 Request anatomy

```http
POST https://contoso-prod.snowflakecomputing.com/api/v2/statements?requestId=<uuid>
Authorization: Bearer <JWT>
X-Snowflake-Authorization-Token-Type: KEYPAIR_JWT
Content-Type: application/json
Accept: application/json

{
  "statement": "SELECT SALE_ID, SALE_DATE, STORE, PRODUCT, QTY, AMOUNT FROM CONTOSO_PROD.API.V_SALES WHERE SALE_DATE >= ? ORDER BY SALE_DATE",
  "bindings": { "1": { "type": "DATE", "value": "2026-09-01" } },
  "timeout": 60,
  "warehouse": "WH_API",
  "role": "R_CLIENT_NORTHWIND"
}
```

| Response | Meaning |
| --- | --- |
| `200` | Done. `data` holds the first partition, and `resultSetMetaData.partitionInfo` lists all partitions |
| `202` | Still running. Poll `GET /api/v2/statements/{statementHandle}` |
| `422` | SQL error. The body has `code` and `message` |
| `429` | Too many requests. Back off and retry |

`requestId` makes retries idempotent: resubmitting with the same ID and `retry=true` won't run the statement twice.

### 3.3 Python client with key-pair JWT

```python
# pip install pyjwt cryptography requests
import base64, hashlib, time, uuid, requests, jwt
from cryptography.hazmat.primitives import serialization

ACCOUNT = "CONTOSO-PROD"                   # org-account, uppercase
USER = "SVC_CLIENT_NORTHWIND"
HOST = "https://contoso-prod.snowflakecomputing.com"

with open("rsa_key.p8", "rb") as f:
    pkey = serialization.load_pem_private_key(f.read(), password=None)
pub_der = pkey.public_key().public_bytes(serialization.Encoding.DER,
                                         serialization.PublicFormat.SubjectPublicKeyInfo)
fp = "SHA256:" + base64.b64encode(hashlib.sha256(pub_der).digest()).decode()

def make_jwt():
    now = int(time.time())
    return jwt.encode({"iss": f"{ACCOUNT}.{USER}.{fp}", "sub": f"{ACCOUNT}.{USER}",
                       "iat": now, "exp": now + 3600}, pkey, algorithm="RS256")

def run(sql, binds):
    H = {"Authorization": f"Bearer {make_jwt()}",
         "X-Snowflake-Authorization-Token-Type": "KEYPAIR_JWT",
         "Content-Type": "application/json", "Accept": "application/json"}
    body = {"statement": sql, "bindings": binds, "timeout": 60,
            "warehouse": "WH_API", "role": "R_CLIENT_NORTHWIND"}
    r = requests.post(f"{HOST}/api/v2/statements", params={"requestId": str(uuid.uuid4())},
                      headers=H, json=body)
    while r.status_code == 202:
        time.sleep(1)
        r = requests.get(f"{HOST}/api/v2/statements/{r.json()['statementHandle']}", headers=H)
    r.raise_for_status()
    res = r.json()
    cols = [c["name"] for c in res["resultSetMetaData"]["rowType"]]
    rows = res["data"]
    handle = res["statementHandle"]
    for i in range(1, len(res["resultSetMetaData"]["partitionInfo"])):
        rows += requests.get(f"{HOST}/api/v2/statements/{handle}",
                             params={"partition": i}, headers=H).json()["data"]
    return [dict(zip(cols, row)) for row in rows]

print(run("SELECT * FROM CONTOSO_PROD.API.V_SALES WHERE SALE_DATE >= ?",
          {"1": {"type": "DATE", "value": "2026-09-01"}}))
```

Values come back as **strings** (dates as epoch-day strings, timestamps as epoch seconds with fractions). Convert them using `rowType[].type`.

### 3.4 OAuth and PAT alternatives

- **External OAuth (Entra ID):** configure a security integration of `TYPE = EXTERNAL_OAUTH`. The client gets an Entra token with scope `session:role:R_CLIENT_NORTHWIND` and sends it with `X-Snowflake-Authorization-Token-Type: OAUTH`.
- **Programmatic access token:** `ALTER USER SVC_CLIENT_NORTHWIND ADD PROGRAMMATIC ACCESS TOKEN pat_northwind ROLE_RESTRICTION = 'R_CLIENT_NORTHWIND' DAYS_TO_EXPIRY = 90;`. Send it as `Authorization: Bearer <token>` with token type `PROGRAMMATIC_ACCESS_TOKEN`. PATs require a network policy on the user.

### 3.5 Lab check

- [ ] The script returns rows with the key-pair JWT.
- [ ] A JWT with `exp` more than 1 hour ahead is rejected.
- [ ] Binding `'1; DROP TABLE X'` as a value returns no rows, with no injection.
- [ ] `QUERY_HISTORY` shows the query with user `SVC_CLIENT_NORTHWIND` and role `R_CLIENT_NORTHWIND`.

## Module 4 — Lab: FastAPI on Snowpark Container Services

SPCS runs your container next to the data. Snowflake handles TLS, ingress authentication and a short-lived database token inside the container, so there are no credentials in your code. Callers must be Snowflake users, which suits partners you onboard as service users.

### 4.1 How identity flows

| Mode | Queries run as | Row access policies see | Configure |
| --- | --- | --- | --- |
| **Owner's rights** (default) | The service's owner role | The service owner | Nothing extra |
| **Caller's rights** | The calling user (restricted by the service's grants) | The **caller** | `capabilities.securityContext.executeAsCaller: true` in the spec, plus caller grants |

Ingress adds the header `Sf-Context-Current-User` (the caller's username) to every request. With caller's rights, it also adds `Sf-Context-Current-User-Token`, which you combine with the service token to open a caller's-rights session.

### 4.2 Application code

```python
# app.py
import os
from datetime import date
import snowflake.connector
from fastapi import FastAPI, Header, HTTPException, Query

app = FastAPI(title="ContosoData Client API (SPCS)")

def service_token() -> str:
    with open("/snowflake/session/token") as f:     # refreshed by Snowflake
        return f.read()

def connect_as_caller(caller_token: str):
    return snowflake.connector.connect(
        host=os.environ["SNOWFLAKE_HOST"], account=os.environ["SNOWFLAKE_ACCOUNT"],
        authenticator="oauth", token=f"{service_token()}.{caller_token}",
        warehouse=os.environ.get("SNOWFLAKE_WAREHOUSE", "WH_API"),
        database="CONTOSO_PROD", schema="API")

@app.get("/health")
def health():
    return {"ok": True}

@app.get("/v1/sales")
def sales(date_from: date, date_to: date, limit: int = Query(1000, le=10000),
          sf_context_current_user: str = Header(None),
          sf_context_current_user_token: str = Header(None)):
    if not sf_context_current_user_token:
        raise HTTPException(401, "Caller's rights not enabled")
    with connect_as_caller(sf_context_current_user_token) as cn:
        cur = cn.cursor(snowflake.connector.DictCursor)
        cur.execute("""SELECT SALE_ID, SALE_DATE, STORE, PRODUCT, QTY, AMOUNT
                       FROM V_SALES WHERE SALE_DATE BETWEEN %(f)s AND %(t)s
                       ORDER BY SALE_DATE LIMIT %(n)s""",
                    {"f": date_from, "t": date_to, "n": limit})
        return cur.fetchall()
```

```dockerfile
FROM python:3.12-slim
WORKDIR /srv
RUN pip install --no-cache-dir fastapi uvicorn snowflake-connector-python
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 4.3 Snowflake objects

```sql
USE ROLE ACCOUNTADMIN;
CREATE ROLE IF NOT EXISTS R_SPCS_OWNER;
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE R_SPCS_OWNER;      -- needed for public endpoints
CREATE COMPUTE POOL API_POOL MIN_NODES = 1 MAX_NODES = 2 INSTANCE_FAMILY = CPU_X64_XS
  AUTO_SUSPEND_SECS = 600;
GRANT USAGE, MONITOR ON COMPUTE POOL API_POOL TO ROLE R_SPCS_OWNER;
GRANT USAGE ON WAREHOUSE WH_API TO ROLE R_SPCS_OWNER;
GRANT USAGE ON DATABASE CONTOSO_PROD TO ROLE R_SPCS_OWNER;
GRANT USAGE, CREATE SERVICE, CREATE IMAGE REPOSITORY ON SCHEMA CONTOSO_PROD.API TO ROLE R_SPCS_OWNER;

USE ROLE R_SPCS_OWNER;
CREATE IMAGE REPOSITORY CONTOSO_PROD.API.REPO;
SHOW IMAGE REPOSITORIES;   -- copy repository_url
```

```bash
docker build --platform linux/amd64 -t client-api:1.0 .
snow spcs image-registry login        # or: docker login <org>-<acct>.registry.snowflakecomputing.com
docker tag client-api:1.0 <repository_url>/client-api:1.0
docker push <repository_url>/client-api:1.0
```

```sql
CREATE SERVICE CONTOSO_PROD.API.CLIENT_API
  IN COMPUTE POOL API_POOL
  QUERY_WAREHOUSE = WH_API
  MIN_INSTANCES = 1 MAX_INSTANCES = 3
  FROM SPECIFICATION $$
spec:
  containers:
    - name: api
      image: /contoso_prod/api/repo/client-api:1.0
      env:
        SNOWFLAKE_WAREHOUSE: WH_API
      readinessProbe: { port: 8000, path: /health }
  endpoints:
    - name: http
      port: 8000
      public: true
  capabilities:
    securityContext:
      executeAsCaller: true
serviceRoles:
  - name: api_user
    endpoints: [http]
$$;

SHOW ENDPOINTS IN SERVICE CONTOSO_PROD.API.CLIENT_API;   -- ingress_url
GRANT SERVICE ROLE CONTOSO_PROD.API.CLIENT_API!api_user TO ROLE R_CLIENT_NORTHWIND;
-- caller's rights also need a caller grant, e.g.:
GRANT CALLER USAGE ON WAREHOUSE WH_API TO ROLE R_SPCS_OWNER;
GRANT CALLER SELECT ON VIEW CONTOSO_PROD.API.V_SALES TO ROLE R_SPCS_OWNER;
```

### 4.4 Calling the public endpoint programmatically

The ingress expects a Snowflake credential in the header `Authorization: Snowflake Token="<token>"`. For a service user, the simplest token is a **programmatic access token** (Module 3.4). Alternatively, exchange a key-pair JWT for an OAuth token scoped to the endpoint.

```python
import requests
r = requests.get("https://<ingress_url>/v1/sales",
                 params={"date_from": "2026-09-01", "date_to": "2026-09-30"},
                 headers={"Authorization": f'Snowflake Token="{PAT}"'})
print(r.status_code, r.json()[:3])
```

The exact programmatic-ingress flow has changed across releases. Follow the current SPCS "public endpoint access from outside Snowflake" page for your account.

### 4.5 Operations

- **Logs:** `SELECT SYSTEM$GET_SERVICE_LOGS('CONTOSO_PROD.API.CLIENT_API', 0, 'api', 100);` or the event table.
- **Status:** `SHOW SERVICE CONTAINERS IN SERVICE ...`, `DESCRIBE SERVICE ...`.
- **Upgrade:** push a new tag, then `ALTER SERVICE ... FROM SPECIFICATION ...` for a rolling update.
- **Cost:** compute pools bill per node-hour while active. Suspend dev pools with `ALTER COMPUTE POOL API_POOL SUSPEND`.
- **Egress:** containers have no internet egress unless you attach an **external access integration** with network rules.

### 4.6 Lab check

- [ ] The endpoint returns 401/redirect without a Snowflake token, and rows with one.
- [ ] Two clients with different roles get different rows (caller's rights + row access policy).
- [ ] `QUERY_HISTORY` shows the **caller's** user name for API queries.

## Module 5 — Lab: Secure Data Sharing, reader accounts and listings

With Secure Data Sharing, a client queries your live data from its own Snowflake account. Nothing is copied, you build no API, and you can revoke access with one statement. One secure view keyed on `CURRENT_ACCOUNT()` serves every client.

### 5.1 Concepts

| Object | What it is |
| --- | --- |
| **Share** | A named set of grants (database, schemas, secure views, tables) given to consumer accounts |
| **Consumer account** | The client's Snowflake account, in the same region (or via a listing, any region) |
| **Reader account** | A managed account *you* create and pay for, for clients without Snowflake |
| **Listing** | A share packaged with title, description, terms, sample queries, usage metrics and cross-region auto-fulfilment. Private (to named accounts) or public (Marketplace) |
| **Database role in a share** | Lets you give different consumers different subsets of the same share |

### 5.2 One secure view for all consumers

```sql
USE ROLE SYSADMIN;
CREATE SCHEMA IF NOT EXISTS CONTOSO_PROD.SHARE_V;

CREATE OR REPLACE TABLE CONTOSO_PROD.SHARE_V.CONSUMER_CLIENT_MAP (
  CONSUMER_ACCOUNT STRING,   -- account locator, as returned by CURRENT_ACCOUNT()
  CLIENT_ID NUMBER);
INSERT INTO CONTOSO_PROD.SHARE_V.CONSUMER_CLIENT_MAP VALUES ('NW12345', 1), ('FB67890', 2);

CREATE OR REPLACE SECURE VIEW CONTOSO_PROD.SHARE_V.V_CLIENT_SALES AS
SELECT s.SALE_ID, s.SALE_DATE, s.STORE, s.PRODUCT, s.QTY, s.AMOUNT
FROM CONTOSO_PROD.GOLD.SALES s
JOIN CONTOSO_PROD.SHARE_V.CONSUMER_CLIENT_MAP m
  ON m.CLIENT_ID = s.CLIENT_ID
 AND m.CONSUMER_ACCOUNT = CURRENT_ACCOUNT();

-- Test as if you were the consumer
ALTER SESSION SET SIMULATED_DATA_SHARING_CONSUMER = 'NW12345';
SELECT * FROM CONTOSO_PROD.SHARE_V.V_CLIENT_SALES;   -- only client 1
ALTER SESSION UNSET SIMULATED_DATA_SHARING_CONSUMER;
```

The view is **secure**, so consumers can't see its definition or the mapping table. The mapping table itself is not granted to the share.

### 5.3 Create the share

```sql
USE ROLE ACCOUNTADMIN;     -- or a role with CREATE SHARE
CREATE SHARE CLIENT_SALES COMMENT = 'Per-client sales, refreshed daily 05:00 UTC';
GRANT USAGE ON DATABASE CONTOSO_PROD TO SHARE CLIENT_SALES;
GRANT USAGE ON SCHEMA CONTOSO_PROD.SHARE_V TO SHARE CLIENT_SALES;
GRANT SELECT ON VIEW CONTOSO_PROD.SHARE_V.V_CLIENT_SALES TO SHARE CLIENT_SALES;

ALTER SHARE CLIENT_SALES ADD ACCOUNTS = FABRIKAMORG.FABRIKAM_PROD;   -- client with Snowflake
```

### 5.4 Reader account for a client without Snowflake

```sql
CREATE MANAGED ACCOUNT NORTHWIND_READER
  ADMIN_NAME = 'northwind_admin', ADMIN_PASSWORD = '<strong-temp-password>',
  TYPE = READER, COMMENT = 'Northwind Ltd – contract C-2026-014';
SHOW MANAGED ACCOUNTS;                       -- note locator and URL
ALTER SHARE CLIENT_SALES ADD ACCOUNTS = NW12345;
```

Then, in the reader account (logged in as its admin), create a small warehouse with a **resource monitor**, because *you* pay for its compute. Create users for the client and grant them the imported database:

```sql
CREATE DATABASE CONTOSO_DATA FROM SHARE <provider_locator>.CLIENT_SALES;
CREATE WAREHOUSE READER_WH WAREHOUSE_SIZE = XSMALL AUTO_SUSPEND = 60;
CREATE RESOURCE MONITOR RM_READER WITH CREDIT_QUOTA = 20 FREQUENCY = MONTHLY
  TRIGGERS ON 90 PERCENT DO SUSPEND;
ALTER WAREHOUSE READER_WH SET RESOURCE_MONITOR = RM_READER;
```

The client can then query through the web UI, ODBC/JDBC, Python or the SQL API, using the reader account.

### 5.5 Listings for scale

Use a **private listing** instead of a raw share when you have many clients or clients in other regions or clouds:

- **Cross-cloud auto-fulfilment** replicates the data product to the consumer's region automatically.
- **Listing access history** (`SNOWFLAKE.DATA_SHARING_USAGE.LISTING_ACCESS_HISTORY`) shows which consumer queried which objects. Plain shares give you no consumer-side query visibility.
- Attach **terms of use** and **sample SQL**. On the Marketplace you can also offer paid listings.

### 5.6 Revoke and monitor

```sql
ALTER SHARE CLIENT_SALES REMOVE ACCOUNTS = NW12345;    -- instant
SHOW GRANTS TO SHARE CLIENT_SALES;
SHOW GRANTS OF SHARE CLIENT_SALES;                       -- which accounts
```

### 5.7 Lab check

- [ ] The simulated consumer sees only its own rows.
- [ ] The reader account can query the share; its monthly credits are capped by the resource monitor.
- [ ] Removing the account makes the consumer's database return an error immediately.
- [ ] (Listing) Access history shows a row for the consumer's first query.

## Module 6 — Lab: external FastAPI + gateway, with semantic views

For a commercial API product, put a gateway (APIM, AWS API Gateway or Apigee) in front of a FastAPI service that connects to Snowflake as a service user. Use **semantic views** so every KPI the API returns is computed the same way as in your dashboards.

### 6.1 Build the semantic view

```sql
CREATE OR REPLACE SEMANTIC VIEW CONTOSO_PROD.API.SV_SALES
  TABLES (
    sales AS CONTOSO_PROD.GOLD.SALES PRIMARY KEY (SALE_ID)
      COMMENT = 'One row per sale line'
  )
  FACTS (
    sales.amount AS AMOUNT,
    sales.qty    AS QTY
  )
  DIMENSIONS (
    sales.sale_id    AS SALE_ID,
    sales.client_id  AS CLIENT_ID,
    sales.sale_year  AS YEAR(SALE_DATE),
    sales.sale_month AS DATE_TRUNC('MONTH', SALE_DATE),
    sales.product    AS PRODUCT
  )
  METRICS (
    sales.total_sales AS SUM(sales.amount),
    sales.total_units AS SUM(sales.qty),
    sales.avg_basket  AS SUM(sales.amount) / COUNT(DISTINCT sales.sale_id)
  )
  COMMENT = 'Governed sales KPIs for clients and dashboards';

-- Query: pick dimensions and metrics; Snowflake generates the right GROUP BY
SELECT * FROM SEMANTIC_VIEW(
    CONTOSO_PROD.API.SV_SALES
    DIMENSIONS sales.sale_month
    METRICS sales.total_sales, sales.total_units, sales.avg_basket
    WHERE sales.client_id = 1 AND sales.sale_year = 2026
) ORDER BY sale_month;
```

Semantic views can also hold relationships between several tables, synonyms and sample values, which Cortex Analyst uses. Check the current DDL reference, since the clause set has grown over releases.

### 6.2 Per-request role switching (database-enforced isolation)

Rather than filtering by `client_id` in Python, give each client its own role, and grant all client roles to the API's service user. For each request, the API runs `USE ROLE R_CLIENT_<id>`, and the **row access policy** (Module 9) filters rows for that role. Even if the code forgets a `WHERE`, the database refuses other clients' rows.

```python
# app/db.py
import os, snowflake.connector
from snowflake.connector import DictCursor

def connect():
    return snowflake.connector.connect(
        account=os.environ["SF_ACCOUNT"],            # contoso-prod
        user="SVC_CLIENT_API",
        authenticator="SNOWFLAKE_JWT",
        private_key_file="/secrets/rsa_key.p8",
        warehouse="WH_API", database="CONTOSO_PROD", schema="API",
        session_parameters={"STATEMENT_TIMEOUT_IN_SECONDS": 120})

ROLE_BY_CLIENT = {1: "R_CLIENT_NORTHWIND", 2: "R_CLIENT_FABRIKAM"}   # load from a table in practice

def query_as_client(client_id: int, sql: str, params: dict):
    role = ROLE_BY_CLIENT[client_id]               # whitelisted identifier, never user input
    with connect() as cn:
        cur = cn.cursor(DictCursor)
        cur.execute(f"USE ROLE {role}")
        cur.execute("ALTER SESSION SET QUERY_TAG = %(t)s", {"t": f"client-api:{client_id}"})
        cur.execute(sql, params)
        return cur.fetchall()
```

```python
# app/main.py
import hmac, os
from datetime import date
from fastapi import FastAPI, Depends, Header, HTTPException, Query
from .db import query_as_client

app = FastAPI(title="ContosoData Client API", version="1.0.0")
KEY_TO_CLIENT = {}   # loaded at startup from CONTOSO_PROD.SEC.API_SUBSCRIPTION

def current_client(x_gateway_secret: str = Header(...), x_client_key_id: str = Header(...)) -> int:
    if not hmac.compare_digest(x_gateway_secret, os.environ["GATEWAY_SECRET"]):
        raise HTTPException(401)
    if x_client_key_id not in KEY_TO_CLIENT:
        raise HTTPException(403)
    return KEY_TO_CLIENT[x_client_key_id]

@app.get("/v1/sales")
def sales(date_from: date, date_to: date, limit: int = Query(1000, le=10000),
          client_id: int = Depends(current_client)):
    return query_as_client(client_id, """
        SELECT SALE_ID, SALE_DATE, STORE, PRODUCT, QTY, AMOUNT FROM V_SALES
        WHERE SALE_DATE BETWEEN %(f)s AND %(t)s ORDER BY SALE_DATE LIMIT %(n)s""",
        {"f": date_from, "t": date_to, "n": limit})

@app.get("/v1/kpis/monthly")
def kpis(year: int, client_id: int = Depends(current_client)):
    return query_as_client(client_id, """
        SELECT * FROM SEMANTIC_VIEW(CONTOSO_PROD.API.SV_SALES
            DIMENSIONS sales.sale_month
            METRICS sales.total_sales, sales.total_units, sales.avg_basket
            WHERE sales.sale_year = %(y)s) ORDER BY sale_month""", {"y": year})
```

The `QUERY_TAG` makes every query attributable to a client in `QUERY_HISTORY` and in cost reports. In production, keep a connection pool rather than connecting per request.

### 6.3 Optional: natural-language endpoint with Cortex Analyst

```python
r = requests.post(f"{HOST}/api/v2/cortex/analyst/message", headers=H, json={
    "messages": [{"role": "user", "content": [{"type": "text", "text": "Sales by month in 2026"}]}],
    "semantic_view": "CONTOSO_PROD.API.SV_SALES"})
sql = next(c["statement"] for c in r.json()["message"]["content"] if c["type"] == "sql")
# execute `sql` with query_as_client(...) so the row access policy still applies
```

Cortex Analyst only *generates* SQL. Always execute it under the client's role so isolation holds, and log the question, SQL and result for review.

### 6.4 Gateway, deploy, secrets

- **Gateway:** API keys + quotas per client product. Inject `X-Gateway-Secret` and `X-Client-Key-Id`, and restrict backend ingress to the gateway (see the Fabric and Databricks courses for full APIM and AWS policies).
- **Secrets:** store the private key in Key Vault, Secrets Manager or Secret Manager and mount it as a file. Better: use **workload identity federation**, so the container's cloud identity authenticates to Snowflake directly, with no key at all.
- **Network policy** on `SVC_CLIENT_API` allowing only the API's egress IPs (Module 11).

### 6.5 Lab check

- [ ] `/v1/kpis/monthly` matches the same semantic-view query run in Snowsight.
- [ ] Removing the `WHERE` on `client_id` in code still returns only that client's rows (thanks to the role + policy).
- [ ] `QUERY_HISTORY` shows `QUERY_TAG = client-api:<id>` for each call.

## Module 7 — Securing the API

The same five rules apply here as on every platform: authenticate every caller, identify each caller, isolate rows in the database, keep secrets out of code, and cap consumption. Snowflake gives you a strong database-side lever for each.

### 7.1 Least privilege per option

| Option | Identity | Minimum grants |
| --- | --- | --- |
| SQL API | `TYPE=SERVICE` user per client | `USAGE` on `WH_API`, the database and `API` schema; `SELECT` on API views; default role = client role |
| SPCS | Service owner role + caller | Owner: `BIND SERVICE ENDPOINT`, compute pool, `CALLER` grants. Caller: service role + client role |
| Secure Data Sharing | Consumer account | Share grants on secure views only |
| External FastAPI | One `SERVICE` user holding all client roles | Each client role: `SELECT` on API views; no other privileges |

Never use `ACCOUNTADMIN`, `SYSADMIN` or `SECURITYADMIN` as the default role of a service user.

### 7.2 Isolation patterns

| Pattern | Enforced by | Strength |
| --- | --- | --- |
| Role per client + row access policy (`IS_ROLE_IN_SESSION`) | Database | Strong |
| Secure view with `CURRENT_ACCOUNT()` mapping (sharing) | Database | Strong |
| Caller's rights in SPCS + row access policy | Database | Strong |
| Code filter with one shared role | Application | Weakest. Needs an automated cross-client test |

### 7.3 Authentication policies

Authentication policies restrict *how* users may log in:

```sql
CREATE AUTHENTICATION POLICY CONTOSO_PROD.SEC.AP_SERVICE
  AUTHENTICATION_METHODS = ('KEYPAIR', 'OAUTH', 'PROGRAMMATIC_ACCESS_TOKEN')
  CLIENT_TYPES = ('DRIVERS', 'SNOWSQL')
  COMMENT = 'Service users: no passwords, no UI';
ALTER USER SVC_CLIENT_API SET AUTHENTICATION POLICY CONTOSO_PROD.SEC.AP_SERVICE;

CREATE AUTHENTICATION POLICY CONTOSO_PROD.SEC.AP_HUMANS
  AUTHENTICATION_METHODS = ('SAML')
  MFA_ENROLLMENT = REQUIRED;
ALTER ACCOUNT SET AUTHENTICATION POLICY CONTOSO_PROD.SEC.AP_HUMANS;   -- account default
```

Check the allowed values (for example the `CLIENT_TYPES` list and MFA options) against current docs.

### 7.4 Network policies per service user

```sql
CREATE NETWORK RULE CONTOSO_PROD.SEC.NR_API_EGRESS
  MODE = INGRESS TYPE = IPV4 VALUE_LIST = ('20.50.10.0/28');      -- your API's outbound IPs
CREATE NETWORK POLICY NP_SVC_CLIENT_API
  ALLOWED_NETWORK_RULE_LIST = ('CONTOSO_PROD.SEC.NR_API_EGRESS');
ALTER USER SVC_CLIENT_API SET NETWORK_POLICY = NP_SVC_CLIENT_API;
```

A user-level policy overrides the account-level one. Stolen keys are useless from any other IP.

### 7.5 Secrets

- **Key-pair:** generate keys per environment. Rotate with the two key slots: set `RSA_PUBLIC_KEY_2`, switch the client, then unset `RSA_PUBLIC_KEY`.
- **PATs:** set short `DAYS_TO_EXPIRY`, restrict them to one role, and require a network policy.
- **Workload identity federation:** no secret at all. Prefer it for anything running in AWS, Azure, GCP or GitHub Actions.
- **Inside Snowflake:** `SECRET` objects plus external access integrations for code that calls outside APIs.

### 7.6 Cost and abuse controls

- A dedicated `WH_API` with a **resource monitor**:

```sql
CREATE RESOURCE MONITOR RM_API WITH CREDIT_QUOTA = 300 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 75 PERCENT DO NOTIFY ON 100 PERCENT DO SUSPEND ON 110 PERCENT DO SUSPEND_IMMEDIATE;
ALTER WAREHOUSE WH_API SET RESOURCE_MONITOR = RM_API;
```

- `STATEMENT_TIMEOUT_IN_SECONDS` and `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` on the warehouse or user.
- Gateway quotas, plus row and date-range caps in code.
- `QUERY_TAG` per client for chargeback.

### 7.7 Threat checklist

| Threat | Control |
| --- | --- |
| Cross-client reads | Role per client + row access policy; secure-view mapping for shares |
| SQL injection | Bind variables (`?` / `%(name)s`); role names only from a whitelist |
| Stolen key or token | Network policy per user; short PAT expiry; key rotation |
| Password logins on service users | `TYPE=SERVICE` + authentication policy |
| Runaway cost | Resource monitor, timeouts, dedicated warehouse |
| Leaking view logic | Secure views and secure UDFs |

## Module 8 — Governance foundations

Snowflake groups its governance features under **Horizon**: access control, policies, tags, classification, lineage, access history and data quality. Everything starts with a clean role design. Most Snowflake governance problems are role sprawl problems.

### 8.1 Role design: access roles vs functional roles

| Layer | Example | Holds | Granted to |
| --- | --- | --- | --- |
| **Access roles** (often database roles) | `CONTOSO_PROD.GOLD_READ`, `CONTOSO_PROD.GOLD_WRITE` | Object privileges on one database or schema | Functional roles |
| **Functional roles** | `R_ANALYST`, `R_DATA_ENGINEER`, `R_CLIENT_NORTHWIND` | Access roles only, never direct object grants | Users / IdP groups (via SCIM) |
| **System roles** | `SYSADMIN`, `SECURITYADMIN` | Top of the hierarchy | Functional roles roll up to `SYSADMIN` |

```sql
USE ROLE SECURITYADMIN;
-- Access roles as DATABASE ROLES (portable, shareable)
USE ROLE SYSADMIN;
CREATE DATABASE ROLE CONTOSO_PROD.GOLD_READ;
GRANT USAGE ON SCHEMA CONTOSO_PROD.GOLD TO DATABASE ROLE CONTOSO_PROD.GOLD_READ;
GRANT SELECT ON ALL TABLES IN SCHEMA CONTOSO_PROD.GOLD TO DATABASE ROLE CONTOSO_PROD.GOLD_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA CONTOSO_PROD.GOLD TO DATABASE ROLE CONTOSO_PROD.GOLD_READ;

-- Functional role
USE ROLE SECURITYADMIN;
CREATE ROLE R_ANALYST;
GRANT USAGE ON DATABASE CONTOSO_PROD TO ROLE R_ANALYST;
GRANT DATABASE ROLE CONTOSO_PROD.GOLD_READ TO ROLE R_ANALYST;
GRANT USAGE ON WAREHOUSE WH_BI TO ROLE R_ANALYST;
GRANT ROLE R_ANALYST TO ROLE SYSADMIN;          -- roll-up
```

**Future grants** make new tables automatically readable by the access role, so access doesn't drift as tables are added.

### 8.2 Managed access schemas

In a normal schema, each object's owner can grant access to anyone. In a **managed access** schema, only the schema owner (or a role with `MANAGE GRANTS`) can grant. This centralises control for sensitive schemas:

```sql
ALTER SCHEMA CONTOSO_PROD.GOLD ENABLE MANAGED ACCESS;
```

### 8.3 Object tagging

```sql
CREATE TAG CONTOSO_PROD.SEC.PII ALLOWED_VALUES 'email', 'phone', 'name', 'iban'
  COMMENT = 'Personal data category';
CREATE TAG CONTOSO_PROD.SEC.DATA_OWNER;
CREATE TAG CONTOSO_PROD.SEC.TIER ALLOWED_VALUES 'bronze', 'silver', 'gold';

ALTER TABLE CONTOSO_PROD.GOLD.SALES SET TAG CONTOSO_PROD.SEC.TIER = 'gold',
                                           CONTOSO_PROD.SEC.DATA_OWNER = 'sales-data-team';
ALTER TABLE CONTOSO_PROD.GOLD.SALES MODIFY COLUMN CUSTOMER_EMAIL SET TAG CONTOSO_PROD.SEC.PII = 'email';

-- Who has which tag (ACCOUNT_USAGE, ~2 h latency)
SELECT object_database, object_schema, object_name, column_name, tag_name, tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE tag_name = 'PII' AND object_deleted IS NULL;
```

- **Tag lineage:** tags on a column propagate to downstream objects when **tag propagation** is enabled on the tag.
- Grant `APPLY TAG` only to a governance role, so tags are trustworthy enough to drive policies (Module 9).

### 8.4 Sensitive data classification

```sql
-- Classify one table and apply system tags (SNOWFLAKE.CORE.SEMANTIC_CATEGORY / PRIVACY_CATEGORY)
CALL SYSTEM$CLASSIFY('CONTOSO_PROD.GOLD.SALES', {'auto_tag': true});

-- Or schedule automatic classification for a whole database with a classification profile
CREATE SNOWFLAKE.DATA_PRIVACY.CLASSIFICATION_PROFILE CONTOSO_PROD.SEC.CP_DEFAULT(
  {'minimum_object_age_for_classification_days': 0, 'maximum_classification_validity_days': 30, 'auto_tag': true});
ALTER DATABASE CONTOSO_PROD SET CLASSIFICATION_PROFILE = 'CONTOSO_PROD.SEC.CP_DEFAULT';
```

Classification can also map its results to **your** tags (for example `PII = 'email'`), so masking policies attached to your tags apply automatically. Review the results: classification is probabilistic.

### 8.5 Discovery: Horizon Catalog

- **Universal search** and the **Horizon Catalog** in Snowsight let users find tables, views, semantic views and listings by name, comment and tag.
- **Object lineage** (Snowsight → object → *Lineage*) shows upstream and downstream objects, built from access history.
- **Internal Marketplace / organizational listings** publish governed data products between accounts in your organisation, with owners and descriptions.
- Write a `COMMENT` on every gold object covering grain, refresh schedule and owner.

### 8.6 Lab

1. Build access roles `GOLD_READ` and `API_READ` as database roles with future grants.
2. Rebuild `R_CLIENT_NORTHWIND` to hold only `API_READ` plus warehouse usage.
3. Enable managed access on `GOLD` and try to grant from a non-owner role. It should fail.
4. Run classification on `GOLD.SALES` and confirm `CUSTOMER_EMAIL` is detected.

## Module 9 — Fine-grained policies

Snowflake policies are schema-level objects that you attach to tables, views or tags. They're evaluated on every query, from every client, including the SQL API, SPCS, shares and BI tools. All need **Enterprise edition** or higher.

| Policy | Controls | Attach to |
| --- | --- | --- |
| Row access policy | Which rows are visible | Table or view (one or more columns) |
| Masking policy | What a column value looks like | Column, or a tag (tag-based masking) |
| Projection policy | Whether a column may appear in the `SELECT` output at all | Column |
| Aggregation policy | Whether rows may be returned only as aggregates over a minimum group size | Table or view |

### 9.1 Row access policy with a mapping table

```sql
USE ROLE SYSADMIN;
CREATE OR REPLACE TABLE CONTOSO_PROD.SEC.ROLE_CLIENT_MAP (ROLE_NAME STRING, CLIENT_ID NUMBER);
INSERT INTO CONTOSO_PROD.SEC.ROLE_CLIENT_MAP VALUES
  ('R_CLIENT_NORTHWIND', 1), ('R_CLIENT_FABRIKAM', 2);

CREATE OR REPLACE ROW ACCESS POLICY CONTOSO_PROD.SEC.RAP_CLIENT
AS (cid NUMBER) RETURNS BOOLEAN ->
     IS_ROLE_IN_SESSION('R_DATA_ENGINEER')
  OR EXISTS (SELECT 1 FROM CONTOSO_PROD.SEC.ROLE_CLIENT_MAP m
             WHERE m.CLIENT_ID = cid AND IS_ROLE_IN_SESSION(m.ROLE_NAME));

ALTER TABLE CONTOSO_PROD.GOLD.SALES ADD ROW ACCESS POLICY CONTOSO_PROD.SEC.RAP_CLIENT ON (CLIENT_ID);

-- Test
USE ROLE R_CLIENT_NORTHWIND;
SELECT DISTINCT CLIENT_ID FROM CONTOSO_PROD.API.V_SALES;   -- 1 only
```

Notes:

- `IS_ROLE_IN_SESSION` also honours **secondary roles** and the role hierarchy. `CURRENT_ROLE()` checks only the primary role.
- The policy owner's privileges are used to read the mapping table, so clients don't need access to `SEC`.
- The same policy can add a sharing branch, `OR EXISTS (... m.CONSUMER_ACCOUNT = CURRENT_ACCOUNT())`, to cover shares.
- Keep mapping lookups simple. Policies run on every query, so very complex subqueries slow everything down.

### 9.2 Masking policies

```sql
CREATE OR REPLACE MASKING POLICY CONTOSO_PROD.SEC.MP_EMAIL
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN IS_ROLE_IN_SESSION('R_PII_READER') THEN val
    WHEN val IS NULL THEN NULL
    ELSE REGEXP_REPLACE(val, '^[^@]+', '***')
  END;

ALTER TABLE CONTOSO_PROD.GOLD.SALES MODIFY COLUMN CUSTOMER_EMAIL
  SET MASKING POLICY CONTOSO_PROD.SEC.MP_EMAIL;
```

**Conditional masking** uses other columns in the decision, for example to unmask only for the client that owns the row:

```sql
CREATE OR REPLACE MASKING POLICY CONTOSO_PROD.SEC.MP_EMAIL_OWNER
AS (val STRING, cid NUMBER) RETURNS STRING ->
  CASE WHEN EXISTS (SELECT 1 FROM CONTOSO_PROD.SEC.ROLE_CLIENT_MAP m
                    WHERE m.CLIENT_ID = cid AND IS_ROLE_IN_SESSION(m.ROLE_NAME)) THEN val
       ELSE '***MASKED***' END;
ALTER TABLE CONTOSO_PROD.GOLD.SALES MODIFY COLUMN CUSTOMER_EMAIL
  SET MASKING POLICY CONTOSO_PROD.SEC.MP_EMAIL_OWNER USING (CUSTOMER_EMAIL, CLIENT_ID);
```

(A column can carry only one masking policy at a time, so unset the first before you set this one.)

### 9.3 Tag-based masking: protect by classification, not by table

Attach the masking policy to the **tag**. Every column carrying the tag, in any table now or later, is masked:

```sql
ALTER TAG CONTOSO_PROD.SEC.PII SET MASKING POLICY CONTOSO_PROD.SEC.MP_EMAIL;
```

Combined with auto-classification (Module 8.4), new PII columns are masked without anyone touching them. One tag can hold one masking policy per data type, so create `MP_PII_STRING`, `MP_PII_NUMBER` and so on as needed. Inside the policy, `SYSTEM$GET_TAG_ON_CURRENT_COLUMN('CONTOSO_PROD.SEC.PII')` lets you vary masking by tag value.

### 9.4 Projection and aggregation policies

Projection: allow filtering *on* a column but never returning it:

```sql
CREATE OR REPLACE PROJECTION POLICY CONTOSO_PROD.SEC.PP_NO_EMAIL
AS () RETURNS PROJECTION_CONSTRAINT ->
  CASE WHEN IS_ROLE_IN_SESSION('R_PII_READER')
       THEN PROJECTION_CONSTRAINT(ALLOW => TRUE)
       ELSE PROJECTION_CONSTRAINT(ALLOW => FALSE) END;
ALTER TABLE CONTOSO_PROD.GOLD.SALES MODIFY COLUMN CUSTOMER_EMAIL
  SET PROJECTION POLICY CONTOSO_PROD.SEC.PP_NO_EMAIL;
```

Aggregation: clients may only see aggregates over groups of at least 5 rows (privacy-preserving analytics):

```sql
CREATE OR REPLACE AGGREGATION POLICY CONTOSO_PROD.SEC.AGG_MIN5
AS () RETURNS AGGREGATION_CONSTRAINT ->
  CASE WHEN IS_ROLE_IN_SESSION('R_DATA_ENGINEER') THEN NO_AGGREGATION_CONSTRAINT()
       ELSE AGGREGATION_CONSTRAINT(MIN_GROUP_SIZE => 5) END;
ALTER TABLE CONTOSO_PROD.GOLD.SALES SET AGGREGATION POLICY CONTOSO_PROD.SEC.AGG_MIN5;
```

This pattern underlies **data clean rooms**, which let two companies join data without either seeing the other's rows.

### 9.5 Auditing your policies

```sql
-- Where is a policy attached?
SELECT * FROM TABLE(CONTOSO_PROD.INFORMATION_SCHEMA.POLICY_REFERENCES(
  POLICY_NAME => 'CONTOSO_PROD.SEC.RAP_CLIENT'));

-- Gold tables WITHOUT a row access policy (account-wide)
SELECT t.table_catalog, t.table_schema, t.table_name
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES p
  ON p.ref_database_name = t.table_catalog AND p.ref_schema_name = t.table_schema
 AND p.ref_entity_name = t.table_name AND p.policy_kind = 'ROW_ACCESS_POLICY'
WHERE t.table_schema = 'GOLD' AND t.deleted IS NULL AND p.policy_name IS NULL;
```

### 9.6 Lab

1. Attach `RAP_CLIENT` and verify both client roles and the engineer role.
2. Replace the column mask with tag-based masking and add a new table with a PII-tagged column. It should be masked automatically.
3. Apply the aggregation policy and confirm `SELECT *` fails for a client role while `GROUP BY STORE` works for groups of 5 or more rows.
4. Run the "gold tables without RAP" audit query and fix any gaps.

## Module 10 — Audit, monitoring, data quality and cost

The shared `SNOWFLAKE` database holds a year of operational history as views in `ACCOUNT_USAGE` (and organisation-wide in `ORGANIZATION_USAGE`). Access history adds column-level read and write auditing. Data metric functions measure quality, and resource monitors and budgets control spend.

### 10.1 Grant governance access without ACCOUNTADMIN

```sql
USE ROLE ACCOUNTADMIN;
CREATE ROLE R_GOVERNANCE;
GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE R_GOVERNANCE;   -- policy, tag, access views
GRANT DATABASE ROLE SNOWFLAKE.SECURITY_VIEWER   TO ROLE R_GOVERNANCE;   -- login, grants, users
GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER      TO ROLE R_GOVERNANCE;   -- metering, storage
GRANT DATABASE ROLE SNOWFLAKE.OBJECT_VIEWER     TO ROLE R_GOVERNANCE;
```

| View (`SNOWFLAKE.ACCOUNT_USAGE`) | Answers |
| --- | --- |
| `QUERY_HISTORY` | Who ran what, when, on which warehouse, how long, how many rows |
| `ACCESS_HISTORY` | Which **columns** each query read or wrote, including base objects behind views |
| `LOGIN_HISTORY` | Logins, auth method, IP, success or failure |
| `GRANTS_TO_ROLES` / `GRANTS_TO_USERS` | Current and historical grants |
| `TAG_REFERENCES`, `POLICY_REFERENCES` | Where tags and policies are attached |
| `WAREHOUSE_METERING_HISTORY`, `METERING_DAILY_HISTORY` | Credits by warehouse and service |
| `QUERY_ATTRIBUTION_HISTORY` | Compute cost attributed to individual queries |

`ACCOUNT_USAGE` views have a latency of up to about 45 minutes to 3 hours, depending on the view. Use `INFORMATION_SCHEMA` table functions for near-real-time checks.

### 10.2 Governance queries you'll reuse

Who read the PII column in the last 7 days:

```sql
SELECT ah.query_start_time, ah.user_name, q.role_name, c.value:"columnName"::string AS column_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
     LATERAL FLATTEN(ah.base_objects_accessed) o,
     LATERAL FLATTEN(o.value:"columns") c
JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q ON q.query_id = ah.query_id
WHERE o.value:"objectName"::string = 'CONTOSO_PROD.GOLD.SALES'
  AND c.value:"columnName"::string = 'CUSTOMER_EMAIL'
  AND ah.query_start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP());
```

API usage and cost per client (using `QUERY_TAG`):

```sql
SELECT SPLIT_PART(q.query_tag, ':', 2) AS client_id,
       DATE_TRUNC('day', q.start_time) AS day,
       COUNT(*) AS queries, SUM(q.rows_produced) AS rows_returned,
       SUM(a.credits_attributed_compute) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY a ON a.query_id = q.query_id
WHERE q.query_tag LIKE 'client-api:%' AND q.start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2 ORDER BY 2 DESC, 4 DESC;
```

New grants of powerful roles:

```sql
SELECT created_on, grantee_name, granted_by, role AS granted_role
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE role IN ('ACCOUNTADMIN', 'SECURITYADMIN', 'SYSADMIN')
  AND created_on >= DATEADD(day, -30, CURRENT_TIMESTAMP()) AND deleted_on IS NULL;
```

Password logins by service users (there should be none):

```sql
SELECT event_timestamp, user_name, first_authentication_factor, client_ip
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE user_name LIKE 'SVC_%' AND first_authentication_factor = 'PASSWORD';
```

### 10.3 Alerts

```sql
CREATE OR REPLACE ALERT CONTOSO_PROD.SEC.ALRT_SCRAPING
  WAREHOUSE = WH_ADMIN
  SCHEDULE = '60 MINUTE'
  IF (EXISTS (
      SELECT 1 FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
      WHERE query_tag LIKE 'client-api:%' AND start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
      GROUP BY query_tag HAVING SUM(rows_produced) > 5000000))
  THEN CALL SYSTEM$SEND_EMAIL('gov_email_int', 'security@contoso.com',
                              'Possible API scraping', 'A client exceeded 5M rows in 24h');
ALTER ALERT CONTOSO_PROD.SEC.ALRT_SCRAPING RESUME;
```

### 10.4 Data quality: data metric functions (DMFs)

```sql
ALTER TABLE CONTOSO_PROD.GOLD.SALES SET DATA_METRIC_SCHEDULE = '60 MINUTE';
ALTER TABLE CONTOSO_PROD.GOLD.SALES ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.NULL_COUNT ON (CLIENT_ID);
ALTER TABLE CONTOSO_PROD.GOLD.SALES ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.FRESHNESS ON (SALE_DATE);
ALTER TABLE CONTOSO_PROD.GOLD.SALES ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.DUPLICATE_COUNT ON (SALE_ID);

-- Custom DMF
CREATE OR REPLACE DATA METRIC FUNCTION CONTOSO_PROD.SEC.NEGATIVE_AMOUNTS(t TABLE(amount NUMBER))
RETURNS NUMBER AS 'SELECT COUNT(*) FROM t WHERE amount < 0';
ALTER TABLE CONTOSO_PROD.GOLD.SALES ADD DATA METRIC FUNCTION CONTOSO_PROD.SEC.NEGATIVE_AMOUNTS ON (AMOUNT);

SELECT * FROM SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS
WHERE table_name = 'SALES' ORDER BY measurement_time DESC;
```

You can add **expectations** to DMFs (for example `NULL_COUNT = 0`) and alert on violations, and quality results show in the Horizon Catalog next to the table.

### 10.5 Cost governance

- **Resource monitors** cap warehouse credits and suspend at thresholds (Module 7.6).
- **Budgets** (account-level and custom) track spend for groups of objects, including serverless features, and notify by email or webhook.
- **Object tags** on warehouses (`cost_center`) plus `QUERY_TAG` give per-team and per-client attribution.
- **Auto-suspend = 60 s** on every warehouse. Audit warehouses that don't have it.

### 10.6 Trust Center

The **Trust Center** (Snowsight → Monitoring) runs scanner packages, such as the **CIS Snowflake Benchmark** and Security Essentials. They flag risks like users without MFA, ACCOUNTADMIN without a network policy, or password-based service users, and give remediation steps. Schedule it and review findings monthly.

### 10.7 Lab

1. Create `R_GOVERNANCE` and run the four governance queries.
2. Build a Snowsight dashboard "Client API" (queries, rows, credits per client per day).
3. Attach three DMFs to `GOLD.SALES` and inject a NULL `CLIENT_ID` to see it detected.
4. Run the Trust Center CIS scanner and fix two findings.

## Module 11 — ALM, identity and network

Production Snowflake should change only through reviewed code run by a CI identity, be reachable only from approved networks, and accept only strong authentication.

### 11.1 Infrastructure and schema as code

| Tool | Manages | Notes |
| --- | --- | --- |
| **Terraform** (`snowflakedb/snowflake` provider) | Account objects: warehouses, roles, grants, users, network and auth policies, databases | Best for platform and RBAC; plan/apply in CI |
| **Snowflake CLI** (`snow`) | Deploys SQL, Snowpark, Streamlit, SPCS, Native Apps; runs scripts | `snow sql -f`, `snow spcs service`, project definition files |
| **Git integration** (`CREATE GIT REPOSITORY`) | Read-only clone of your repo inside Snowflake | `EXECUTE IMMEDIATE FROM @repo/branches/main/deploy.sql` |
| **Declarative DCM / `CREATE OR ALTER`** | Idempotent object definitions | Re-runnable deploy scripts without drop/recreate |
| **schemachange**, dbt, Flyway | Versioned migrations / transformation models | Pick one per layer |

Example: deploy from GitHub Actions with workload identity federation (no secret stored):

```yaml
# .github/workflows/deploy.yml
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snowflakedb/snowflake-cli-action@v1
      - run: snow sql -f deploy/api_views.sql --temporary-connection
        env:
          SNOWFLAKE_ACCOUNT: contoso-prod
          SNOWFLAKE_USER: SVC_CICD
          SNOWFLAKE_AUTHENTICATOR: WORKLOAD_IDENTITY
          SNOWFLAKE_WORKLOAD_IDENTITY_PROVIDER: OIDC
```

```sql
-- The CI user trusts GitHub's OIDC tokens for this repo and branch only
CREATE USER SVC_CICD TYPE = SERVICE
  WORKLOAD_IDENTITY = (TYPE = OIDC
    ISSUER = 'https://token.actions.githubusercontent.com'
    SUBJECT = 'repo:contoso/snowflake-platform:ref:refs/heads/main')
  DEFAULT_ROLE = R_CICD;
```

Treat the action names and parameters as illustrative and check the current Snowflake CLI and workload-identity docs before you rely on them.

### 11.2 Environments

- Use separate **accounts** for dev and prod (strongest isolation), or at least separate databases (`CONTOSO_DEV`, `CONTOSO_PROD`) with separate roles.
- **Zero-copy clone** prod into test for realistic testing: `CREATE DATABASE CONTOSO_TEST CLONE CONTOSO_PROD;`. Policies and tags come with the clone, and you must still restrict who can read it.
- **Replication and failover groups** (Business Critical) replicate databases, roles, grants, policies and users to a secondary account for disaster recovery.

### 11.3 Identity

| Control | How | Recommendation |
| --- | --- | --- |
| **SSO** | SAML2 security integration (Entra ID, Okta) | Mandatory for people |
| **SCIM** | SCIM security integration | Users and groups (as roles) provisioned from the IdP |
| **MFA** | Authentication policies with `MFA_ENROLLMENT = REQUIRED` | Snowflake is phasing out single-factor password sign-in for people |
| **Service users** | `TYPE = SERVICE` | Key-pair, OAuth or federation only |
| **Admin roles** | `ACCOUNTADMIN` granted to at least 2 and at most 4 named people, MFA, no default role | Use `SECURITYADMIN` / `SYSADMIN` day to day |
| **Session policies** | `CREATE SESSION POLICY ... SESSION_IDLE_TIMEOUT_MINS = 30` | Shorter for admins |

### 11.4 Network

| Control | What it does |
| --- | --- |
| **Network rules + network policies** (account or user) | Allow or block client IPs, VPC endpoint IDs or Private Link IDs |
| **AWS PrivateLink / Azure Private Link / GCP Private Service Connect** (Business Critical) | Private connectivity to Snowflake, with public access blockable |
| **Private connectivity for internal stages** | Stage traffic over private endpoints |
| **External access integrations** | Explicit allow-list of hosts that UDFs, procedures and SPCS may call out to (anti-exfiltration) |
| **Outbound private connectivity** | Snowflake reaches your services privately |

```sql
-- Account baseline: office + VPN + API egress only
CREATE NETWORK RULE CONTOSO_PROD.SEC.NR_CORP MODE = INGRESS TYPE = IPV4
  VALUE_LIST = ('203.0.113.0/24', '198.51.100.10/32');
CREATE NETWORK POLICY NP_ACCOUNT ALLOWED_NETWORK_RULE_LIST = ('CONTOSO_PROD.SEC.NR_CORP');
ALTER ACCOUNT SET NETWORK_POLICY = NP_ACCOUNT;    -- test with a non-admin user first!
```

### 11.5 Encryption and compliance

- All data is encrypted at rest with hierarchical keys, rotated automatically. **Periodic rekeying** is available on Enterprise and above.
- **Tri-Secret Secure** (Business Critical) combines a Snowflake key with **your** cloud KMS key. Revoking yours makes the data unreadable.
- **Business Critical / VPS** editions support HIPAA, PCI-DSS and FedRAMP workloads.
- **Data residency:** choose the account region. Cross-region sharing needs listings or replication.

### 11.6 Lab

1. Put warehouses, roles and grants for this course in Terraform and apply from CI.
2. Create `SVC_CICD` with workload identity federation and deploy `api_views.sql` from GitHub Actions.
3. Apply the account network policy (after testing on a single user) and the service-user authentication policy.
4. Clone `CONTOSO_PROD` to `CONTOSO_TEST` and verify the row access policy still applies in the clone.

## Module 12 — Capstone, quiz and cheat sheet

### 12.1 Capstone project

**Brief:** onboard three clients in one day. *Tailspin* wants a REST API with a key, *Woodgrove* runs Snowflake in another region, and *Litware* has no Snowflake but wants SQL access.

- [ ] Tailspin: role `R_CLIENT_TAILSPIN`, mapped in `ROLE_CLIENT_MAP`, served by the external FastAPI with per-request role switching
- [ ] Woodgrove: private listing with cross-region auto-fulfilment, secure view keyed on `CURRENT_ACCOUNT()`
- [ ] Litware: reader account with a resource monitor capped at 20 credits/month
- [ ] Tag-based masking on all PII, with classification auto-tagging
- [ ] Authentication policy and network policy on every service user
- [ ] Everything deployed from Git via CI with workload identity federation
- [ ] Snowsight dashboard of per-client usage and cost; scraping alert active
- [ ] One-page data-sharing record per client

**Assessment:** the reviewer tries to read another client's rows through each channel, log in as a service user with a password, call from an unlisted IP, and read `CUSTOMER_EMAIL` unmasked. All must fail.

### 12.2 Knowledge check

1. Why can't you add a normal (non-secure) view to a share?
2. What does `CURRENT_ACCOUNT()` return inside a shared secure view?
3. Who pays for queries in a reader account, and how do you cap that cost?
4. Which user `TYPE` should integrations use, and which auth methods does it allow?
5. Why prefer `IS_ROLE_IN_SESSION()` over `CURRENT_ROLE()` in policies?
6. What does tag-based masking add over column masking?
7. What does an aggregation policy enforce?
8. Which view tells you which columns a query actually read through a view?
9. How do you attribute warehouse cost to individual API clients?
10. What must be in place before a programmatic access token can be used?

**Answers**

1. Only secure objects can be shared, so the consumer can't see the definition or bypass filtering through optimiser behaviour.
2. The **consumer's** account locator, which is why it can drive per-client row filtering.
3. The provider (you). Cap it with a resource monitor on the reader account's warehouse.
4. `TYPE = SERVICE`: key-pair, OAuth, workload identity federation or PATs. No password and no MFA prompt.
5. It includes secondary roles and inherited roles, so the policy behaves correctly in more session setups.
6. New columns are protected automatically when they receive the tag (manually or through classification), with no per-table work.
7. Results must be aggregated over groups of at least N rows. Individual rows can't be returned.
8. `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` (`base_objects_accessed`).
9. Set a `QUERY_TAG` per client on each session, and join `QUERY_HISTORY` to `QUERY_ATTRIBUTION_HISTORY`.
10. A network policy on the user (by default), plus an expiry and ideally a role restriction.

### 12.3 Cheat sheet

| Need | Use |
| --- | --- |
| Client on Snowflake wants live tables | Secure Data Sharing / private listing |
| Client without Snowflake wants SQL access | Reader account (you pay; resource monitor) |
| Simple REST SQL | SQL API with key-pair JWT |
| Custom API next to the data | Snowpark Container Services (caller's rights) |
| Paid API product | External FastAPI + gateway; role per client |
| Same KPIs everywhere | Semantic views; Cortex Analyst for natural language |
| Hide rows | Row access policy (+ `CURRENT_ACCOUNT()` for shares) |
| Hide or transform columns | Masking policy; tag-based masking; projection policy |
| Aggregate-only access | Aggregation policy / clean rooms |
| Who read what | `ACCESS_HISTORY` |
| Data quality | Data metric functions + expectations |
| Cost | Resource monitors, budgets, `QUERY_TAG`, `QUERY_ATTRIBUTION_HISTORY` |
| Security posture | Trust Center (CIS benchmark) |
| Safe change | Terraform + Snowflake CLI + Git integration + workload identity federation |
| Private access | Network policies, Private Link, Tri-Secret Secure |

### Further reading

Check each feature's edition requirements and preview status before you deliver this course. The links below follow the usual Snowflake docs paths but weren't individually checked for this draft.

- [Snowflake SQL API](https://docs.snowflake.com/en/developer-guide/sql-api/index)
- [Snowpark Container Services](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview)
- [Secure Data Sharing](https://docs.snowflake.com/en/user-guide/data-sharing-intro)
- [Semantic views](https://docs.snowflake.com/en/user-guide/views-semantic/overview)
- [Row access policies](https://docs.snowflake.com/en/user-guide/security-row-intro)
- [Masking policies](https://docs.snowflake.com/en/user-guide/security-column-ddm-intro)
- [Access history](https://docs.snowflake.com/en/user-guide/access-history)
- [Data metric functions](https://docs.snowflake.com/en/user-guide/data-quality-intro)
- [Authentication policies](https://docs.snowflake.com/en/user-guide/authentication-policies)
