# Google BigQuery — Data APIs & Governance: Full Course

> Source of truth: the live Claude Doc (https://claude.ai/code/artifact/3ebc1a91-a77d-4bec-8a87-9f7f10763dca), which also holds the drawn diagrams. Exported 2026-10-13. Written from model knowledge; preview-era syntax is flagged in the text for verification.


Oct 13, 2026 · @Matt

## Course overview

By the end of this course you can serve BigQuery data and governed metrics to external clients through the BigQuery API, a Cloud Run service behind API Gateway or Apigee, or Analytics Hub sharing. You'll also govern the platform with IAM, row and column security, Dataplex, audit logs, VPC Service Controls and cost controls.

**Audience:** data engineers, cloud platform engineers and analytics leads who know SQL and Python and have used the Google Cloud console.

**Prerequisites:** a Google Cloud organisation (needed for org policies and VPC Service Controls), a project with billing, the roles *BigQuery Admin*, *Security Admin* and *Service Account Admin* (or a friendly admin), the `gcloud` and `bq` CLIs, and Python 3.10+.

**Running scenario.** *ContosoData* sells retail sales analytics. Each client must get only its own rows and the same KPI definitions used internally. Data lives in project `contoso-data-prod`, dataset `gold`.

### Learning outcomes

1. Explain organisation, folder, project, dataset and table, and where IAM, policies and perimeters apply.
2. Choose between the BigQuery API, a Cloud Run API, Analytics Hub, the Storage Read API and Looker for a client need.
3. Build each option with service accounts or workload identity federation, without long-lived keys.
4. Isolate clients with row access policies and protect columns with policy tags and dynamic masking.
5. Govern with Dataplex Universal Catalog, audit logs, data quality scans, budgets and reservations.
6. Ship changes with Terraform and Dataform behind org policies and VPC Service Controls.

### Module map

| # | Module | Type | Time |
| --- | --- | --- | --- |
| 1 | Architecture essentials | Concepts | 45 min |
| 2 | API options compared | Concepts | 30 min |
| 3 | BigQuery REST API | Lab | 60 min |
| 4 | FastAPI on Cloud Run + gateway | Lab | 2 h |
| 5 | Analytics Hub sharing | Lab | 60 min |
| 6 | Semantic layer + bulk extracts | Lab | 90 min |
| 7 | Securing the API | Concepts + lab | 60 min |
| 8 | Governance foundations | Concepts + lab | 60 min |
| 9 | Fine-grained access | Lab | 90 min |
| 10 | Audit, monitoring, cost | Lab | 60 min |
| 11 | ALM, identity, network | Concepts + lab | 60 min |
| 12 | Capstone + quiz | Project | 3 h |

### Shared setup script

```bash
gcloud config set project contoso-data-prod
bq --location=EU mk --dataset contoso-data-prod:gold
bq --location=EU mk --dataset contoso-data-prod:api
bq --location=EU mk --dataset contoso-data-prod:sec
```

```sql
CREATE OR REPLACE TABLE `contoso-data-prod.gold.sales` (
  sale_id INT64, client_id INT64, sale_date DATE, store STRING,
  product STRING, qty INT64, amount NUMERIC, customer_email STRING)
PARTITION BY sale_date CLUSTER BY client_id;

INSERT INTO `contoso-data-prod.gold.sales` VALUES
 (1,1,'2026-09-01','Paris-01','Coffee',3,12.60,'a@x.com'),
 (2,1,'2026-09-02','Paris-02','Tea',1,4.20,'b@x.com'),
 (3,2,'2026-09-01','Lyon-01','Coffee',5,21.00,'c@y.com');

CREATE OR REPLACE VIEW `contoso-data-prod.api.v_sales` AS
SELECT sale_id, client_id, sale_date, store, product, qty, amount
FROM `contoso-data-prod.gold.sales`;
```

Partitioning by date and clustering by `client_id` keeps per-client queries cheap. BigQuery bills on-demand queries by bytes scanned. Google renames products often: Analytics Hub is now *BigQuery sharing*, and Dataplex Catalog is now *Dataplex Universal Catalog*. Confirm the current names and GA status before you teach.

## Module 1 — BigQuery architecture essentials

BigQuery is serverless: there are no clusters to secure, only **projects**, **datasets** and **tables** governed by Google Cloud IAM. The project you *run* a query in (it pays for compute) can differ from the project that *stores* the data. That separation is the key to client APIs and cost attribution.

### 1.1 The hierarchy

| Level | What it is | Key controls |
| --- | --- | --- |
| **Organization** | Root, tied to your Cloud Identity / Workspace domain | Org policies, org-level IAM, VPC Service Controls access policy |
| **Folder** | Groups projects (by env, domain) | Inherited IAM and org policies |
| **Project** | Billing and quota unit; runs jobs | IAM, APIs enabled, quotas, budgets, reservations assignment |
| **Dataset** | Container of tables, views, routines, models, in **one location** | Dataset IAM, default encryption key, authorized views/routines, default table expiry |
| **Table / view** | Data or logic | Table IAM, row access policies, policy tags (columns), data masking |

Objects are addressed as `project.dataset.table`, for example `` `contoso-data-prod.gold.sales` ``.

### 1.2 Two permissions every query needs

| Need | Typical role | Granted on |
| --- | --- | --- |
| Run a query job | `roles/bigquery.jobUser` (`bigquery.jobs.create`) | The **project that runs** the job (and pays) |
| Read the data | `roles/bigquery.dataViewer` | The **dataset/table** that holds the data |
| Read via the Storage Read API | `roles/bigquery.readSessionUser` | The project that runs the session |

This split lets you put each client in **its own job project**. Their queries are billed to and quota-limited in that project, while the data stays in yours.

### 1.3 Identities

| Identity | Use | Notes |
| --- | --- | --- |
| Google account / group | People | SSO through Cloud Identity / Workspace or a federated IdP |
| **Service account (SA)** | Workloads, APIs | Avoid downloadable **keys**; use attached SAs or impersonation |
| **Workload Identity Federation (WIF)** | External workloads (AWS, Azure, GitHub, a client's IdP via OIDC/SAML) | Exchange an external token for short-lived Google credentials; no SA key |
| **Workforce Identity Federation** | External *people* (client analysts) with their own IdP | Lets them use the console and BigQuery without Google accounts |

### 1.4 Compute and pricing

- **On-demand:** pay per TiB scanned. It's simple, but one bad client query can scan a lot. Cap it with `maximumBytesBilled` and custom quotas.
- **Capacity (editions: Standard, Enterprise, Enterprise Plus):** buy **slots** in **reservations** and assign projects to them. This gives predictable cost and isolation, for example a dedicated reservation for client API projects. Several governance features (such as row-level security on some paths, column masking and VPC-SC integration) depend on edition. Check the edition feature matrix.
- **BI Engine** caches hot data in memory for sub-second dashboard and API queries.

### 1.5 Locations

A dataset lives in one region or multi-region (`EU`, `US`, `europe-west1`…). Queries can't join across locations, and shares are location-bound, so choose the location per data-residency requirement and keep related datasets together.

### 1.6 Exercise

1. Run the setup script.
2. Create a client job project `contoso-client-northwind` under folder `clients`.
3. Create SA `sa-northwind@contoso-client-northwind.iam.gserviceaccount.com`. Grant it `bigquery.jobUser` on its own project and `bigquery.dataViewer` on dataset `api` (Module 3 tightens this to the view only).

## Module 2 — API options compared

BigQuery *is* an API: every query, load and read is a REST or gRPC call. So the question is less "how do I build an API" and more "which interface do I expose, and who pays for the compute".

### 2.1 Side-by-side comparison

| Criterion | BigQuery REST API / client libraries | Cloud Run FastAPI + API Gateway / Apigee | Analytics Hub (BigQuery sharing) | Storage Read API | Looker semantic layer |
| --- | --- | --- | --- | --- | --- |
| Hosted in | Google (managed) | Cloud Run (serverless) | Google (managed) | Google (managed) | Looker instance |
| Protocol | REST/JSON (`jobs.query`) | REST you design | Subscriber queries a linked dataset in its own project | gRPC, Arrow/Avro streams | Looker API (REST), SQL interface, connectors |
| Client auth | Google identity (SA, WIF, Workforce IF) | API keys, JWT, OAuth — your choice | Google identity of the subscriber | Google identity | Looker API credentials / OAuth |
| Who pays queries | Project that runs the job (can be the client's) | You | **Subscriber** | Project that creates the read session | You |
| Quotas, keys, portal | GCP quotas only | Yes (Apigee: full API management) | Per-listing access, egress controls | GCP quotas | Looker roles |
| Best for | Technical clients on GCP | Commercial API products | Clients wanting live tables | High-throughput bulk extracts | Governed KPIs, embedded analytics |

### 2.2 What each option is

**BigQuery REST API and client libraries.** The client calls `jobs.query` (or `jobs.insert` + `getQueryResults`) with a parameterised SQL string, as its own Google identity, in its own or your project. There's no server for you to build. You only grant access to views.

**Cloud Run FastAPI + API Gateway or Apigee.** Your service runs as a service account and queries BigQuery. **API Gateway** adds API keys and basic quotas cheaply. **Apigee** adds products, developer portal, monetisation, analytics and advanced policies.

**Analytics Hub (BigQuery sharing)** publishes **listings** in **data exchanges**. A subscriber gets a read-only **linked dataset** in its own project and pays for its own queries. You can share views and tables, apply **egress controls** (no export or copy), and run **data clean rooms** with aggregation thresholds.

**Storage Read API** streams table data in parallel as Arrow or Avro. It's fast and cheap for bulk extracts (millions of rows) into pandas, Spark or other warehouses, and it respects row and column security.

**Looker** holds the semantic layer in **LookML** (dimensions, measures, joins, access filters). Clients or apps get governed numbers through the Looker API (`run_inline_query`), Looker's SQL interface, or connectors such as Looker Studio, Sheets and Power BI. BigQuery also offers a conversational analytics API on top of curated data, for natural-language access.

### 2.3 Decision tree

&#91;embedded content: BigQuery API option decision tree · 4 questions, 5 outcomes\]

Stop at the first "yes". Every route keeps working only if row access policies and policy tags are on the underlying tables (Module 9), because they apply whichever interface the client uses.

### 2.4 Common mistakes

- **Handing out service-account JSON keys** to clients. Use Workload Identity Federation, or keep identities inside your API.
- **Granting `dataViewer` on the whole project.** Grant on the `api` dataset, or better on individual views (authorized views, Module 9).
- **No `maximumBytesBilled`.** A `SELECT *` over an unpartitioned table can cost hundreds of dollars on-demand.
- **Mixing locations.** An EU dataset can't be joined with a US dataset, and an EU listing can't be subscribed to from US.

## Module 3 — Lab: BigQuery REST API for external clients

In this lab, Northwind's system (running in Azure, outside Google Cloud) queries your view directly through the BigQuery REST API. It authenticates with **Workload Identity Federation**, so no service-account key ever leaves Google. Queries run, and are billed, in Northwind's own job project.

### 3.1 Grant access to one view only

```sql
-- Table-level IAM on the view (not the dataset)
GRANT `roles/bigquery.dataViewer`
ON TABLE `contoso-data-prod.api.v_sales`
TO "serviceAccount:sa-northwind@contoso-client-northwind.iam.gserviceaccount.com";
```

A view normally needs its readers to also have access to the underlying table. To avoid granting `gold.sales`, make `api.v_sales` an **authorized view** on the `gold` dataset (Module 9.4). The view can then read the table on the reader's behalf.

```bash
# Job permissions in the client's own project (they pay for compute)
gcloud projects add-iam-policy-binding contoso-client-northwind \
  --member="serviceAccount:sa-northwind@contoso-client-northwind.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"
```

### 3.2 Workload Identity Federation for an Azure-hosted client

```bash
# 1. Pool + provider trusting the client's Entra ID tenant
gcloud iam workload-identity-pools create clients --location=global
gcloud iam workload-identity-pools providers create-oidc northwind-entra \
  --location=global --workload-identity-pool=clients \
  --issuer-uri="https://sts.windows.net/<northwind-tenant-id>/" \
  --allowed-audiences="api://contoso-bigquery" \
  --attribute-mapping="google.subject=assertion.sub" \
  --attribute-condition="assertion.sub=='<object-id-of-northwind-managed-identity>'"

# 2. Let that external identity impersonate the client SA
gcloud iam service-accounts add-iam-policy-binding \
  sa-northwind@contoso-client-northwind.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principal://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/clients/subject/<object-id>"

# 3. Generate a credential config file for the client (contains NO secret)
gcloud iam workload-identity-pools create-cred-config \
  projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/clients/providers/northwind-entra \
  --service-account=sa-northwind@contoso-client-northwind.iam.gserviceaccount.com \
  --azure --app-id-uri=api://contoso-bigquery --output-file=northwind-wif.json
```

Northwind sets `GOOGLE_APPLICATION_CREDENTIALS=northwind-wif.json`. Google client libraries then exchange their Azure managed-identity token for short-lived Google credentials automatically. The same pattern works for AWS (`--aws`), GitHub Actions and any OIDC or SAML IdP.

### 3.3 Raw REST call

```http
POST https://bigquery.googleapis.com/bigquery/v2/projects/contoso-client-northwind/queries
Authorization: Bearer <google-access-token>
Content-Type: application/json

{
  "query": "SELECT sale_id, sale_date, store, product, qty, amount FROM `contoso-data-prod.api.v_sales` WHERE sale_date >= @from ORDER BY sale_date",
  "useLegacySql": false,
  "parameterMode": "NAMED",
  "queryParameters": [
    { "name": "from", "parameterType": { "type": "DATE" }, "parameterValue": { "value": "2026-09-01" } }
  ],
  "location": "EU",
  "timeoutMs": 30000,
  "maxResults": 1000,
  "maximumBytesBilled": "1000000000",
  "labels": { "client": "northwind", "app": "direct-api" }
}
```

| Response field | Meaning |
| --- | --- |
| `jobComplete` | `false` means poll `GET .../queries/{jobId}?location=EU` |
| `rows[].f[].v` | Cell values (strings; cast with `schema.fields[].type`) |
| `pageToken` | Pass it to `getQueryResults` for the next page |
| `totalBytesProcessed` | Bytes scanned (on-demand cost driver) |

`maximumBytesBilled` makes the job fail instead of scanning more than 1 GB. `labels` flow into billing export and `INFORMATION_SCHEMA.JOBS` for attribution.

### 3.4 Python client library

```python
# pip install google-cloud-bigquery
from google.cloud import bigquery

client = bigquery.Client(project="contoso-client-northwind", location="EU")   # uses WIF creds
job_config = bigquery.QueryJobConfig(
    query_parameters=[bigquery.ScalarQueryParameter("from", "DATE", "2026-09-01")],
    maximum_bytes_billed=1_000_000_000,
    labels={"client": "northwind"})
rows = client.query(
    "SELECT * FROM `contoso-data-prod.api.v_sales` WHERE sale_date >= @from",
    job_config=job_config).result()            # handles polling and pagination
for r in rows:
    print(dict(r))
```

For large results, add `pip install google-cloud-bigquery-storage` and call `rows.to_dataframe()`. The library then switches to the Storage Read API automatically.

### 3.5 Lab check

- [ ] Northwind can query `api.v_sales` but gets *Access Denied* on `gold.sales`.
- [ ] The job appears in **Northwind's** project job history and billing, not yours.
- [ ] With `maximumBytesBilled = 1`, the query fails with *bytesBilledLimitExceeded*.
- [ ] The row access policy (Module 9) limits Northwind to `client_id = 1`.

## Module 4 — Lab: FastAPI on Cloud Run behind API Gateway or Apigee

This is the API-product pattern on Google Cloud. A gateway authenticates clients and enforces quotas. A private Cloud Run service runs FastAPI and queries BigQuery. For database-enforced isolation, the service **impersonates a per-client service account** on each request, so BigQuery row access policies apply.

### 4.1 Choose the gateway

|  | API Gateway | Apigee (X / hybrid) |
| --- | --- | --- |
| Client auth | API keys, Google/Firebase/Auth0/any-OIDC JWT | API keys, OAuth 2.0 (client credentials), JWT, mTLS |
| Quotas | Per-key quota via service config | Quota, SpikeArrest per product/app/developer |
| Developer portal, monetisation | No | Yes |
| Analytics | Basic (Cloud Monitoring) | Rich API analytics per developer/app |
| Cost | Low, pay per call | Higher; subscription or pay-as-you-go |
| Use when | Few clients, simple keys | API is a product with many clients |

### 4.2 Service code

```python
# app/main.py
import os, hmac
from datetime import date
from functools import lru_cache
from fastapi import FastAPI, Depends, Header, HTTPException, Query
import google.auth
from google.auth import impersonated_credentials
from google.cloud import bigquery

app = FastAPI(title="ContosoData Client API", version="1.0.0")
BILLING_PROJECT = os.environ["BILLING_PROJECT"]          # contoso-api-prod
source_creds, _ = google.auth.default()                     # Cloud Run's service account

# client_id -> per-client SA (load from a table / Secret Manager in practice)
CLIENT_SA = {1: "sa-northwind@contoso-client-northwind.iam.gserviceaccount.com",
             2: "sa-fabrikam@contoso-client-fabrikam.iam.gserviceaccount.com"}
KEY_TO_CLIENT = {"northwind-app": 1, "fabrikam-app": 2}     # Apigee app name -> client

@lru_cache(maxsize=256)
def bq_for(client_id: int) -> bigquery.Client:
    creds = impersonated_credentials.Credentials(
        source_credentials=source_creds, target_principal=CLIENT_SA[client_id],
        target_scopes=["https://www.googleapis.com/auth/bigquery"], lifetime=3600)
    return bigquery.Client(project=BILLING_PROJECT, credentials=creds, location="EU")

def current_client(x_gateway_secret: str = Header(...), x_client_app: str = Header(...)) -> int:
    if not hmac.compare_digest(x_gateway_secret, os.environ["GATEWAY_SECRET"]):
        raise HTTPException(401)
    if x_client_app not in KEY_TO_CLIENT:
        raise HTTPException(403)
    return KEY_TO_CLIENT[x_client_app]

def run(client_id: int, sql: str, params: list):
    cfg = bigquery.QueryJobConfig(query_parameters=params, maximum_bytes_billed=2_000_000_000,
                                  labels={"app": "client-api", "client": str(client_id)})
    return [dict(r) for r in bq_for(client_id).query(sql, job_config=cfg).result()]

@app.get("/v1/sales")
def sales(date_from: date, date_to: date, limit: int = Query(1000, le=10000),
          client_id: int = Depends(current_client)):
    if (date_to - date_from).days > 366:
        raise HTTPException(400, "Max 366 days")
    return run(client_id, """
        SELECT sale_id, sale_date, store, product, qty, amount
        FROM `contoso-data-prod.api.v_sales`
        WHERE sale_date BETWEEN @f AND @t ORDER BY sale_date LIMIT @n""",
        [bigquery.ScalarQueryParameter("f", "DATE", date_from),
         bigquery.ScalarQueryParameter("t", "DATE", date_to),
         bigquery.ScalarQueryParameter("n", "INT64", limit)])
```

No `client_id` filter appears in the SQL. The query runs **as the client's SA**, and the row access policy (`SESSION_USER()` mapping, Module 9) returns only that client's rows. The Cloud Run SA needs `roles/iam.serviceAccountTokenCreator` on each client SA, and `bigquery.jobUser` on `contoso-api-prod`. Each client SA needs `jobUser` there too, plus `dataViewer` on the view.

### 4.3 Deploy privately

```bash
gcloud run deploy client-api --source . --region europe-west1 \
  --service-account sa-client-api@contoso-api-prod.iam.gserviceaccount.com \
  --no-allow-unauthenticated --ingress internal-and-cloud-load-balancing \
  --set-env-vars BILLING_PROJECT=contoso-api-prod \
  --set-secrets GATEWAY_SECRET=gateway-secret:latest \
  --min-instances 1 --max-instances 10
```

`--no-allow-unauthenticated` means only identities with `roles/run.invoker` (the gateway's SA) can call the service.

### 4.4 Option A: API Gateway with API keys

```yaml
# openapi.yaml (Swagger 2.0, as API Gateway requires)
swagger: "2.0"
info: { title: contoso-client-api, version: "1.0.0" }
schemes: [https]
x-google-backend:
  address: https://client-api-xxxxx-ew.a.run.app
  jwt_audience: https://client-api-xxxxx-ew.a.run.app
securityDefinitions:
  api_key: { type: apiKey, name: x-api-key, in: header }
x-google-management:
  metrics:
    - { name: "read-requests", displayName: "Read requests", valueType: INT64, metricKind: DELTA }
  quota:
    limits:
      - { name: "read-limit", metric: "read-requests", unit: "1/min/{project}", values: { STANDARD: 600 } }
paths:
  /v1/sales:
    get:
      operationId: listSales
      security: [ { api_key: [] } ]
      x-google-quota: { metricCosts: { read-requests: 1 } }
      responses: { "200": { description: OK } }
```

API Gateway quotas are per **consumer project** of the API key. To identify each client individually, create each client's key in a separate project, or use JWT authentication (the gateway forwards claims in `X-Apigateway-Api-Userinfo`). Your backend then maps the claims to a client.

### 4.5 Option B: Apigee (recommended for API products)

1. Create an **API proxy** targeting the Cloud Run URL, using Google ID-token authentication to the backend.
2. Create **API products** `Bronze` and `Gold` with quotas, and one **developer app** per client. Each app gets a consumer key.
3. Proxy request flow:

```xml
<!-- VerifyAPIKey -->
<VerifyAPIKey name="VK-Verify"><APIKey ref="request.header.x-api-key"/></VerifyAPIKey>
<!-- Quota from the product -->
<Quota name="Q-Product" type="calendar"><UseQuotaConfigInAPIProduct stepName="VK-Verify"/></Quota>
<SpikeArrest name="SA-Burst"><Rate>20ps</Rate></SpikeArrest>
<!-- Forward identity to the backend -->
<AssignMessage name="AM-Identity">
  <Set><Headers>
    <Header name="X-Client-App">{verifyapikey.VK-Verify.app.name}</Header>
    <Header name="X-Gateway-Secret">{private.gateway_secret}</Header>
  </Headers></Set>
  <Remove><Headers><Header name="x-api-key"/></Headers></Remove>
</AssignMessage>
```

### 4.6 Lab check

- [ ] A direct call to the Cloud Run URL returns 403 (no invoker role).
- [ ] Through the gateway, two client keys return disjoint data, even though the SQL has no `client_id` filter.
- [ ] Exceeding the quota returns 429.
- [ ] `INFORMATION_SCHEMA.JOBS` shows jobs run by each **client SA**, labelled `client=<id>`.

## Module 5 — Lab: Analytics Hub (BigQuery sharing)

Analytics Hub lets a client **subscribe** to your data and get a read-only *linked dataset* in its own Google Cloud project. There's no copy, the data is always current, the subscriber pays for its own queries, and you can stop sharing at any time. It's the BigQuery equivalent of Delta Sharing and Snowflake Secure Data Sharing.

### 5.1 Concepts

| Object | What it is |
| --- | --- |
| **Data exchange** | A container of listings, private or public, with its own IAM (publishers, subscribers, admins) |
| **Listing** | A reference to one shared dataset, with description, documentation, contact and restrictions |
| **Linked dataset** | What the subscriber gets in its project: a read-only symbolic link to your dataset |
| **Egress controls** | Listing-level restrictions: block export, copy and query-result egress out of the linked dataset |
| **Data clean room** | A special exchange where shared views carry **analysis rules** (aggregation threshold, list overlap) |

### 5.2 One dataset per client

A listing shares a whole dataset, so give each client a share dataset containing authorized views filtered to it:

```sql
CREATE SCHEMA `contoso-data-prod.share_northwind` OPTIONS (location = 'EU');

CREATE OR REPLACE VIEW `contoso-data-prod.share_northwind.sales` AS
SELECT sale_id, sale_date, store, product, qty, amount
FROM `contoso-data-prod.gold.sales`
WHERE client_id = 1;
```

Then authorise the share dataset on `gold`, so its views can read the table without subscribers having access to `gold` (Console: `gold` → *Sharing* → *Authorize datasets* → add `share_northwind`). Authorising the **dataset** covers every view in it, now and later.

An alternative for many clients: share one dataset and rely on **row access policies keyed on `SESSION_USER()`** (Module 9). It's less isolated and harder to audit than per-client datasets, so it suits large public listings rather than contractual B2B feeds.

### 5.3 Create the exchange and listing (Terraform)

```hcl
resource "google_bigquery_analytics_hub_data_exchange" "clients" {
  project          = "contoso-data-prod"
  location         = "EU"
  data_exchange_id = "contoso_clients"
  display_name     = "ContosoData client feeds"
}

resource "google_bigquery_analytics_hub_listing" "northwind" {
  project          = "contoso-data-prod"
  location         = "EU"
  data_exchange_id = google_bigquery_analytics_hub_data_exchange.clients.data_exchange_id
  listing_id       = "northwind_sales"
  display_name     = "Northwind – daily sales"
  description      = "Per-client sales, refreshed 05:00 UTC. Contract C-2026-014."
  bigquery_dataset { dataset = "projects/contoso-data-prod/datasets/share_northwind" }
  restricted_export_config {
    enabled               = true     # no export / copy out of the linked dataset
    restrict_query_result = true     # query results can't be written to tables outside
  }
}

resource "google_bigquery_analytics_hub_listing_iam_member" "northwind_sub" {
  project          = "contoso-data-prod"
  location         = "EU"
  data_exchange_id = google_bigquery_analytics_hub_data_exchange.clients.data_exchange_id
  listing_id       = google_bigquery_analytics_hub_listing.northwind.listing_id
  role             = "roles/analyticshub.subscriber"
  member           = "group:bq-subscribers@northwind.com"
}
```

Egress restrictions limit what the subscriber can do with results, for example creating tables from them. Check how the current restriction options interact with your clients' BI tools before you enable them.

### 5.4 What the client does

1. In BigQuery Studio → **Analytics Hub** → search or open the listing link → **Subscribe**, choosing a project and a linked-dataset name (for example `contoso_sales`).
2. Query it like any dataset, billed to their project:

```sql
SELECT DATE_TRUNC(sale_date, MONTH) AS month, SUM(amount) AS sales
FROM `northwind-analytics.contoso_sales.sales`
GROUP BY month ORDER BY month;
```

They can also connect Looker Studio, Power BI, Tableau or the BigQuery API to the linked dataset.

### 5.5 Clean room variant (aggregate-only sharing)

When clients may see only aggregates, never single customers, use a **data clean room** and give the shared view an aggregation-threshold analysis rule:

```sql
CREATE OR REPLACE VIEW `contoso-data-prod.cleanroom.sales_agg`
OPTIONS (privacy_policy = '{"aggregation_threshold_policy": {"threshold": 5, "privacy_unit_columns": ["customer_email"]}}')
AS SELECT store, product, sale_date, amount, customer_email FROM `contoso-data-prod.gold.sales`;
```

Subscribers must then use `SELECT WITH AGGREGATION_THRESHOLD ...`, and groups with fewer than 5 distinct customers are suppressed.

### 5.6 Monitor and revoke

- **Subscriptions** on the listing show who subscribed. Remove the subscriber's IAM binding, or **revoke the subscription**, to cut access immediately.
- Publishers can see subscriber usage in the listing's usage metrics and in `INFORMATION_SCHEMA.SHARED_DATASET_USAGE` (the exact view name has changed over releases, so check before you build alerts).

### 5.7 Lab check

- [ ] Northwind subscribes and sees only its rows.
- [ ] An attempt to `CREATE TABLE ... AS SELECT` from the linked dataset into another dataset is blocked by egress controls.
- [ ] Revoking the subscription makes the linked dataset unusable.
- [ ] Usage metrics show Northwind's queries.

## Module 6 — Lab: semantic layer (Looker) and bulk extracts (Storage Read API)

Two client needs fall outside "query a view". **Governed KPIs** must match your dashboards exactly; Looker's LookML model provides them. **Bulk extracts** of millions of rows are too slow and costly through `jobs.query` pagination; the Storage Read API streams them in parallel.

### 6.1 Define KPIs once in LookML

```lookml
# views/sales.view.lkml
view: sales {
  sql_table_name: `contoso-data-prod.gold.sales` ;;

  dimension: sale_id   { primary_key: yes type: number sql: ${TABLE}.sale_id ;; }
  dimension: client_id { type: number sql: ${TABLE}.client_id ;; hidden: yes }
  dimension: product   { type: string sql: ${TABLE}.product ;; }
  dimension_group: sale {
    type: time
    timeframes: [date, month, year]
    datatype: date
    sql: ${TABLE}.sale_date ;;
  }

  measure: total_sales { type: sum sql: ${TABLE}.amount ;; value_format_name: eur }
  measure: total_units { type: sum sql: ${TABLE}.qty ;; }
  measure: orders      { type: count_distinct sql: ${sale_id} ;; }
  measure: avg_basket  { type: number sql: ${total_sales} / NULLIF(${orders}, 0) ;; value_format_name: eur }
}

# models/contoso.model.lkml
connection: "bq_contoso_prod"
include: "/views/*.view.lkml"

explore: sales {
  access_filter: {
    field: sales.client_id
    user_attribute: client_id      # each Looker user/embed user carries their client_id
  }
}
```

The `access_filter` is Looker's row-level security: every query a user runs in this Explore gets `WHERE client_id = <their attribute>` added automatically.

### 6.2 Serve KPIs through the Looker API

Give each client a Looker **API user** with the user attribute `client_id`, a role limited to the `contoso` model, and API3 credentials. They (or your FastAPI facade) call:

```python
# pip install looker-sdk ; env: LOOKERSDK_BASE_URL, LOOKERSDK_CLIENT_ID, LOOKERSDK_CLIENT_SECRET
import looker_sdk
from looker_sdk import models40 as m

sdk = looker_sdk.init40()
result = sdk.run_inline_query(
    result_format="json",
    body=m.WriteQuery(
        model="contoso", view="sales",
        fields=["sales.sale_month", "sales.total_sales", "sales.total_units", "sales.avg_basket"],
        filters={"sales.sale_year": "2026"},
        sorts=["sales.sale_month"], limit="500"))
print(result)
```

If your own API calls Looker on clients' behalf, authenticate once as a service admin and use `sdk.auth.login_user(<client user id>)` (*sudo*) per request, so the client's `access_filter` applies. Alternatively, Looker's **Open SQL Interface** (JDBC) lets BI tools query LookML Explores as if they were tables.

Other semantic options on Google Cloud: Dataform-managed views, **materialized views** for pre-aggregated KPIs, and the BigQuery conversational analytics API, which uses curated metadata for natural-language questions.

### 6.3 Bulk extracts with the Storage Read API

```python
# pip install google-cloud-bigquery-storage pyarrow
from google.cloud.bigquery_storage import BigQueryReadClient, types

client = BigQueryReadClient()
session = client.create_read_session(
    parent="projects/contoso-client-northwind",                # billing project
    read_session=types.ReadSession(
        table="projects/contoso-data-prod/datasets/gold/tables/sales",
        data_format=types.DataFormat.ARROW,
        read_options=types.ReadSession.TableReadOptions(
            selected_fields=["sale_id", "sale_date", "store", "product", "qty", "amount"],
            row_restriction="sale_date >= '2026-01-01'")),
    max_stream_count=4)

frames = []
for stream in session.streams:                                  # parallelise in production
    reader = client.read_rows(stream.name)
    frames.append(reader.to_arrow(session).to_pandas())
```

Key facts:

- The Storage Read API reads **tables** (and materialized views or snapshots), not logical views. Expose a table, or let the client run a query whose results land in a temporary table.
- **Row access policies, column-level security and masking are enforced** for the reading identity, so isolation still holds.
- It's billed per bytes read, which is usually cheaper than query scanning for full extracts. `selected_fields` and `row_restriction` push projection and filters down.

### 6.4 Lab check

- [ ] Two Looker API users with different `client_id` attributes get different totals for the same query.
- [ ] The Looker `avg_basket` for 2026 equals `SUM(amount) / COUNT(DISTINCT sale_id)` computed in BigQuery, not the average of monthly averages.
- [ ] A Storage Read API extract as Northwind's SA returns only client 1 rows and no `customer_email` column.

## Module 7 — Securing the API

The five rules again: authenticate every caller, identify each caller, isolate rows in the database, keep secrets out of code, and cap consumption. On Google Cloud the big wins are **no service-account keys**, **one identity per client** and **cost caps per project or job**.

### 7.1 Least privilege per option

| Option | Identity | Minimum grants |
| --- | --- | --- |
| Direct REST API | Client SA (via WIF) | `bigquery.dataViewer` on specific **views**; `bigquery.jobUser` on the client's job project |
| Cloud Run API | Run SA + per-client SAs | Run SA: `iam.serviceAccountTokenCreator` on client SAs, `jobUser` on the API project. Client SAs: `dataViewer` on views |
| Analytics Hub | Subscriber group | `analyticshub.subscriber` on one listing |
| Storage Read API | Client SA | `dataViewer` on the table + `bigquery.readSessionUser` on the billing project |
| Looker | Looker API user | Looker role on one model + `client_id` user attribute |

Avoid basic roles (`Owner`, `Editor`, `Viewer`) everywhere. They include broad BigQuery permissions across the project.

### 7.2 No keys: org policies to enforce it

```bash
# Block creation and upload of SA keys across the organisation
gcloud resource-manager org-policies enable-enforce iam.disableServiceAccountKeyCreation --organization=<ORG_ID>
gcloud resource-manager org-policies enable-enforce iam.disableServiceAccountKeyUpload  --organization=<ORG_ID>
# Only allow identities from your domain(s) in IAM policies (domain-restricted sharing)
# constraints/iam.allowedPolicyMemberDomains -> your Cloud Identity customer IDs (+ partner IDs if needed)
```

Domain-restricted sharing blocks granting access to *external* Google accounts. If clients use their own Google identities (for example as Analytics Hub subscribers), add their customer IDs as exceptions, or use folder-level policy overrides for sharing projects.

### 7.3 Isolation options

| Pattern | Enforced by | Strength |
| --- | --- | --- |
| Per-client SA + row access policy on `SESSION_USER()` | BigQuery | Strong |
| Per-client share dataset with authorized views | BigQuery IAM | Strong, easy to audit |
| Looker `access_filter` | Looker | Strong within Looker; BigQuery access must be Looker-only |
| One SA, code filter | Application | Weakest. Needs automated cross-client tests |

### 7.4 Cost and abuse controls

- **Per-job cap:** `maximum_bytes_billed` on every API query.
- **Custom quotas** (IAM & Admin → Quotas → BigQuery API): *Query usage per day* (per project) and *Query usage per day per user*. These cap on-demand TiB scanned per client SA.
- **Reservations:** assign client and API projects to a dedicated reservation with a slot cap, so they can't consume internal capacity.
- **Partitioning + clustering + `require_partition_filter = TRUE`**, so clients can't scan the whole table accidentally:

```sql
ALTER TABLE `contoso-data-prod.gold.sales` SET OPTIONS (require_partition_filter = TRUE);
```

- **Gateway quotas** (Apigee or API Gateway) for request rates.
- **Job labels** (`client`, `app`) for chargeback through billing export.

### 7.5 Secrets

- Prefer **attached service accounts** (Cloud Run, GKE Workload Identity) and **Workload Identity Federation** for external callers. There are no secrets to store.
- Keep everything else (gateway shared secret, Looker API credentials) in **Secret Manager**, mounted with `--set-secrets` and rotated with versions.

### 7.6 Threat checklist

| Threat | Control |
| --- | --- |
| Cross-client reads | Per-client identity + row access policy / per-client authorized views |
| SQL injection | Query parameters (`@name`) only; never string formatting |
| Leaked SA key | Org policy blocks keys; WIF with attribute conditions |
| Data exfiltration to another project | **VPC Service Controls** perimeter around data projects (Module 11) |
| Runaway on-demand cost | `maximum_bytes_billed`, custom quotas, partition filters, reservations |
| Direct access to the backend | Cloud Run `--no-allow-unauthenticated` + internal ingress |

### 7.7 Exercise

Write a pytest that impersonates each client SA, runs `SELECT DISTINCT client_id FROM api.v_sales`, and asserts exactly one value equal to the expected client. Run it in CI after every deployment.

## Module 8 — Governance foundations

On Google Cloud, governance is spread across three layers: the **resource hierarchy + IAM** (who can do what where), **Dataplex Universal Catalog** (what data exists, what it means, where it came from), and **policies** (org policies, row and column security, perimeters). This module covers the first two.

### 8.1 Resource hierarchy design

```
Organization: contoso.com
├── folder: data-platform
│   ├── folder: prod
│   │   ├── project: contoso-data-prod        ← storage: gold, api, share_* datasets
│   │   └── project: contoso-api-prod         ← Cloud Run, gateway, API job billing
│   └── folder: nonprod
│       └── project: contoso-data-dev
└── folder: clients                          ← one job project per client (billing + quota)
    ├── project: contoso-client-northwind
    └── project: contoso-client-fabrikam
```

- **Separate storage projects from compute projects.** Data access is granted on datasets, while cost and quotas are controlled per compute project.
- **Folders** carry org policies and IAM that differ between environments (for example stricter policies in `prod`).
- **Labels** on projects and datasets (`env`, `domain`, `cost_center`, `client`) flow into billing export.

### 8.2 IAM design

| Principle | Practice |
| --- | --- |
| Groups, not people | Bind roles to Google Groups / Cloud Identity groups synced from your IdP |
| Narrowest resource | Prefer table/view or dataset bindings over project bindings |
| Predefined roles first | `bigquery.dataViewer`, `dataEditor`, `jobUser`, `user`, `dataOwner`, `metadataViewer` |
| Custom roles when needed | For example a `bqApiReader` role with only `bigquery.tables.getData` + `bigquery.tables.get` |
| Conditions | IAM conditions on **resource tags** or time, for example access only to datasets tagged `env=dev` |
| Just-in-time admin | Privileged Access Manager (PAM) grants with approval and expiry |

```bash
# Dataset-level grant to a group
bq add-iam-policy-binding --member=group:analysts@contoso.com \
  --role=roles/bigquery.dataViewer contoso-data-prod:gold

# Tag-based conditional access: analysts read every dataset tagged sensitivity=internal
gcloud resource-manager tags keys create sensitivity --parent=organizations/<ORG_ID>
gcloud resource-manager tags values create internal --parent=<ORG_ID>/sensitivity
```

### 8.3 Four kinds of "tags": don't confuse them

| Mechanism | Attached to | Purpose |
| --- | --- | --- |
| **Labels** | Projects, datasets, tables, jobs | Free-form key/value for billing and filtering. No access control |
| **Resource tags** (Resource Manager) | Org, folders, projects, datasets, tables | Governed key/values that **IAM conditions and org policies** can reference |
| **Policy tags** (Data Catalog taxonomies) | Columns | **Column-level security and dynamic masking** (Module 9) |
| **Dataplex aspects** | Catalog entries (tables, columns, other assets) | Rich business metadata: owner, domain, data-product info, quality status |

### 8.4 Dataplex Universal Catalog

- **Automatic discovery:** BigQuery datasets, tables, views and models (plus Cloud Storage, Spanner, Cloud SQL and others) appear as **entries** with technical metadata, with no setup.
- **Aspects and aspect types:** you define a schema (for example `data_product` with `owner`, `domain`, `tier`, `sla_hours`, `certified`) and attach it to entries. Aspects are searchable and readable through the API.
- **Business glossary:** terms like *Net Sales* and *Active Client*, linked to columns.
- **Search:** semantic and keyword search in BigQuery Studio and the Dataplex console, filtered by aspects, projects and labels.
- **Data lineage:** captured automatically for BigQuery jobs (and Dataflow, Composer, Dataproc and others when the Data Lineage API is enabled). It's shown as a graph on each table and queryable through the Lineage API.
- **Data profiling and quality scans** attach statistics and quality results to entries (Module 10).

Example: attach a certification aspect with the `gcloud` CLI (the exact syntax depends on your aspect type):

```bash
gcloud dataplex entries update \
  projects/contoso-data-prod/locations/eu/entryGroups/@bigquery/entries/<entry-id> \
  --update-aspects=aspects.yaml
```

```yaml
# aspects.yaml
contoso-data-prod.eu.data-product:
  data:
    owner: sales-data-team@contoso.com
    domain: sales
    tier: gold
    certified: true
    exposed_via: [client-api, analytics-hub]
```

### 8.5 Descriptions

Set descriptions in DDL so they live with the code:

```sql
ALTER TABLE `contoso-data-prod.gold.sales` SET OPTIONS (
  description = 'One row per sale line. Grain: sale_id. Refreshed 05:00 UTC. Owner: sales-data-team.',
  labels = [('domain', 'sales'), ('tier', 'gold')]);
ALTER TABLE `contoso-data-prod.gold.sales` ALTER COLUMN customer_email
  SET OPTIONS (description = 'Customer email. PII. Masked for non-privileged readers.');
```

### 8.6 Lab

1. Build the folder and project layout above, or a mini version of it in a sandbox.
2. Replace any project-level `dataViewer` grants with dataset or table grants to groups.
3. Create a `data-product` aspect type and attach it to `gold.sales`.
4. Open the lineage graph for `api.v_sales` and confirm it shows `gold.sales` upstream.

## Module 9 — Fine-grained access

BigQuery has four fine-grained tools: **row access policies** (rows), **policy tags** (column-level access), **data policies** (dynamic masking) and **authorized views, datasets and routines** (controlled exposure without granting base tables). **Sensitive Data Protection** finds what needs protecting.

### 9.1 Row access policies

Static policy, one per client:

```sql
CREATE OR REPLACE ROW ACCESS POLICY rap_northwind
ON `contoso-data-prod.gold.sales`
GRANT TO ('serviceAccount:sa-northwind@contoso-client-northwind.iam.gserviceaccount.com',
          'group:northwind-analysts@contoso.com')
FILTER USING (client_id = 1);

CREATE OR REPLACE ROW ACCESS POLICY rap_engineers
ON `contoso-data-prod.gold.sales`
GRANT TO ('group:data-engineers@contoso.com')
FILTER USING (TRUE);
```

Dynamic policy from a mapping table, which scales to many clients:

```sql
CREATE OR REPLACE TABLE `contoso-data-prod.sec.principal_client` (principal STRING, client_id INT64);
INSERT INTO `contoso-data-prod.sec.principal_client` VALUES
  ('sa-northwind@contoso-client-northwind.iam.gserviceaccount.com', 1),
  ('sa-fabrikam@contoso-client-fabrikam.iam.gserviceaccount.com', 2);

CREATE OR REPLACE ROW ACCESS POLICY rap_clients_dynamic
ON `contoso-data-prod.gold.sales`
GRANT TO ('group:api-clients@contoso.com')     -- all client SAs are members
FILTER USING (client_id IN (
  SELECT client_id FROM `contoso-data-prod.sec.principal_client`
  WHERE principal = SESSION_USER()));
```

Rules to remember:

- Once a table has **any** row access policy, users not in any policy's grantee list see **zero rows**.
- Policies are combined with OR: a user in two policies sees the union.
- Policies apply through views, the Storage Read API, Analytics Hub linked datasets and BI tools.
- Managing them needs `bigquery.rowAccessPolicies.*` (in `bigquery.admin` / `dataOwner`). Keep that with the platform team.

### 9.2 Column-level security with policy tags

1. **Create a taxonomy** (BigQuery → *Policy tags*, or Data Catalog API) named `contoso-sensitivity`, with tags `PII > Email`, `PII > Phone`, `Financial > IBAN`. Turn on **Enforce access control**.
2. **Grant** `roles/datacatalog.categoryFineGrainedReader` on the `Email` tag to `group:pii-readers@contoso.com`.
3. **Attach** the tag to the column:

```bash
bq show --schema --format=prettyjson contoso-data-prod:gold.sales > schema.json
# edit schema.json: on customer_email add
#   "policyTags": {"names": ["projects/contoso-data-prod/locations/eu/taxonomies/<TAX_ID>/policyTags/<EMAIL_TAG_ID>"]}
bq update contoso-data-prod:gold.sales schema.json
```

Users without the fine-grained reader role get *Access Denied* when they select `customer_email`. They can still query the other columns with `SELECT * EXCEPT (customer_email)`.

### 9.3 Dynamic data masking

Instead of denying access, show masked values. Create a **data policy** on the policy tag:

```bash
curl -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" \
  "https://bigquerydatapolicy.googleapis.com/v1/projects/contoso-data-prod/locations/eu/dataPolicies" \
  -d '{
    "dataPolicyId": "mask_email",
    "dataPolicyType": "DATA_MASKING_POLICY",
    "policyTag": "projects/contoso-data-prod/locations/eu/taxonomies/<TAX_ID>/policyTags/<EMAIL_TAG_ID>",
    "dataMaskingPolicy": { "predefinedExpression": "EMAIL_MASK" }
  }'
```

Then grant `roles/bigquerydatapolicy.maskedReader` on that data policy to the readers who should see masked values.

| Reader has | Sees |
| --- | --- |
| Fine-grained reader on the tag | Real value |
| Masked reader on the data policy | Masked value (for example `XXXXX@x.com`) |
| Neither | Access denied on that column |

Predefined masking rules include `SHA256`, `ALWAYS_NULL`, `DEFAULT_MASKING_VALUE`, `EMAIL_MASK`, `LAST_FOUR_CHARACTERS`, `FIRST_FOUR_CHARACTERS` and `DATE_YEAR_MASK`. Custom masking routines (UDFs) are also supported. Masking needs Enterprise edition or above under capacity pricing; check current edition requirements.

### 9.4 Authorized views, datasets and routines

An **authorized view** reads source tables with the view's authorisation instead of the reader's, so readers need access only to the view:

```bash
# Authorize the api dataset (all its views) to read gold
bq show --format=prettyjson contoso-data-prod:gold > gold.json
# add to "access": {"dataset": {"dataset": {"projectId": "contoso-data-prod", "datasetId": "api"}, "targetTypes": ["VIEWS"]}}
bq update --source gold.json contoso-data-prod:gold
```

- **Authorized dataset:** every view in `api` is authorised, including future ones. It's the cleanest pattern for an API contract dataset.
- **Authorized routine (UDF/TVF):** a table function such as `api.sales_for(from_date)` can encapsulate logic and filters. Readers get `EXECUTE` but no table access.
- Row access policies on the source table **still apply** when the data is read through an authorized view.

### 9.5 Find sensitive data: Sensitive Data Protection

- **Discovery** (data profiles) scans BigQuery across the organisation, folder or project on a schedule, and records the infoTypes found per column (`EMAIL_ADDRESS`, `IBAN_CODE`, `PERSON_NAME`), plus risk and sensitivity scores.
- Profiles can be published to Dataplex as aspects and to Security Command Center. You can also auto-tag resources so that IAM conditions apply.
- **De-identification** jobs (tokenise, pseudonymise, redact) produce safe copies for test environments or sharing.

### 9.6 Lab

1. Create the dynamic row access policy, then verify with each client SA and an engineer.
2. Create the taxonomy, attach `Email` to `customer_email`, and test as a user without the fine-grained role.
3. Add the `EMAIL_MASK` data policy and test as a masked reader.
4. Turn `api` into an authorized dataset and remove all direct grants on `gold` for client identities.
5. Run an SDP discovery scan on the project and review the profile for `gold.sales`.

## Module 10 — Audit, monitoring, data quality and cost

BigQuery audit data comes from two places. **Cloud Audit Logs** record who did what (admin changes and data access). **`INFORMATION_SCHEMA`** views record jobs, bytes, slots and objects. Route both into a governance dataset and query them with SQL.

### 10.1 Cloud Audit Logs

| Log | Contents | On by default |
| --- | --- | --- |
| **Admin Activity** | Dataset/table creation, IAM changes, policy changes | Yes, can't be disabled |
| **Data Access** | Reads: `jobs.insert` queries, `tabledata.list`, Storage Read API sessions | For BigQuery: **yes** (BigQuery data access logs are on by default, unlike most services) |
| **System Event** | Google-initiated changes | Yes |
| **Policy Denied** | Requests blocked by VPC Service Controls | Yes |

Centralise them with an **aggregated log sink** at organisation level into a BigQuery dataset in a locked-down security project:

```bash
gcloud logging sinks create org-bq-audit \
  bigquery.googleapis.com/projects/contoso-security/datasets/audit_logs \
  --organization=<ORG_ID> --include-children \
  --log-filter='protoPayload.serviceName="bigquery.googleapis.com" OR protoPayload.serviceName="bigquerydatapolicy.googleapis.com" OR protoPayload.serviceName="analyticshub.googleapis.com"' \
  --use-partitioned-tables
# then grant the sink's writer identity roles/bigquery.dataEditor on the audit_logs dataset
```

### 10.2 Governance queries you'll reuse

IAM changes on BigQuery datasets in the last 30 days:

```sql
SELECT timestamp, protoPayload.authenticationInfo.principalEmail AS actor,
       protoPayload.resourceName AS resource, protoPayload.methodName AS method
FROM `contoso-security.audit_logs.cloudaudit_googleapis_com_activity`
WHERE protoPayload.serviceName = 'bigquery.googleapis.com'
  AND protoPayload.methodName LIKE '%SetIamPolicy%'
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
ORDER BY timestamp DESC;
```

Client API usage and cost per client (job labels):

```sql
SELECT (SELECT value FROM UNNEST(labels) WHERE key = 'client') AS client,
       DATE(creation_time) AS day,
       COUNT(*) AS queries,
       SUM(total_bytes_billed) / POW(1024, 4) AS tib_billed,
       SUM(total_slot_ms) / 1000 / 3600 AS slot_hours
FROM `region-eu`.INFORMATION_SCHEMA.JOBS_BY_ORGANIZATION
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND EXISTS (SELECT 1 FROM UNNEST(labels) WHERE key = 'app' AND value = 'client-api')
GROUP BY client, day ORDER BY day DESC, tib_billed DESC;
```

Who read the PII column (jobs referencing the table, joined to the user):

```sql
SELECT j.creation_time, j.user_email, j.query
FROM `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT j, UNNEST(j.referenced_tables) t
WHERE t.dataset_id = 'gold' AND t.table_id = 'sales'
  AND REGEXP_CONTAINS(LOWER(j.query), r'customer_email')
  AND j.creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY);
```

`JOBS` has no column-level access record, so text matching is approximate. For precise column access, use the Data Access audit log's `metadata.tableDataRead.fields`, where your release provides it.

### 10.3 Alerts

- Create **log-based metrics** and Cloud Monitoring alerts, for example on any `SetIamPolicy` on `contoso-data-prod` by a principal other than the CI SA, or any VPC-SC *Policy Denied* event.
- **Scheduled queries** that write anomalies (more than 1 TiB per client per day; a new principal reading `gold`) to a table, with an alert on new rows.

### 10.4 Data quality: Dataplex scans

```yaml
# dq_sales.yaml – Dataplex data quality scan rules
rules:
  - column: client_id
    nonNullExpectation: {}
    dimension: COMPLETENESS
  - column: sale_id
    uniquenessExpectation: {}
    dimension: UNIQUENESS
  - column: amount
    rangeExpectation: { minValue: "0" }
    dimension: VALIDITY
  - tableConditionExpectation:
      sqlExpression: "MAX(sale_date) >= DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)"
    dimension: FRESHNESS
```

```bash
gcloud dataplex datascans create data-quality dq-gold-sales --location=europe-west1 \
  --data-source-resource="//bigquery.googleapis.com/projects/contoso-data-prod/datasets/gold/tables/sales" \
  --data-quality-spec-file=dq_sales.yaml --schedule="0 6 * * *"
```

Results are published to the table's Dataplex entry and can be exported to BigQuery. **Data profile scans** add distributions and null rates automatically.

### 10.5 Cost governance

- **Cloud Billing export** to BigQuery (detailed usage) with labels, giving cost by project, dataset label and job label.
- **Budgets and alerts** per project or folder, with Pub/Sub notifications. They can trigger automation, for example lowering a client's custom quota.
- **Reservations:** baseline + autoscaling slots per edition; assign client projects to a capped reservation.
- **Custom quotas** per project and per user (Module 7.4).
- **Recommender** suggestions: partitioning/clustering, unused tables, slot right-sizing.

### 10.6 Lab

1. Create the organisation log sink and run the IAM-change query.
2. Build a Looker Studio report "Client API" from `JOBS_BY_ORGANIZATION` grouped by the `client` label.
3. Create the data quality scan, insert a NULL `client_id`, and see the scan fail.
4. Set a monthly budget on `contoso-client-northwind` with alerts at 50/90/100%.

## Module 11 — ALM, identity and network

On Google Cloud the strongest anti-exfiltration control is **VPC Service Controls**: a perimeter that stops BigQuery data leaving your projects even when an identity has valid permissions. Combine it with infrastructure as code, keyless CI and org policies.

### 11.1 Infrastructure and transformations as code

| Tool | Manages |
| --- | --- |
| **Terraform** (`google` provider) | Projects, IAM, datasets, tables, row access policies (where supported), reservations, Analytics Hub, VPC-SC, org policies |
| **Dataform** (in BigQuery) | SQLX transformation models, assertions (data tests), dependencies, release configs per environment, Git-backed |
| **dbt** (alternative) | The same role as Dataform, run from your CI or Cloud Run jobs |
| **Cloud Build / GitHub Actions** | CI/CD running `terraform apply` and Dataform/dbt with WIF identities |

Example Dataform model with assertions:

```sqlx
-- definitions/api/v_sales.sqlx
config {
  type: "view",
  schema: "api",
  description: "Client API contract view – no PII",
  assertions: { nonNull: ["sale_id", "client_id", "sale_date"], uniqueKey: ["sale_id"] }
}
SELECT sale_id, client_id, sale_date, store, product, qty, amount
FROM ${ref("sales")}
```

Keyless deploy from GitHub Actions:

```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: actions/checkout@v4
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: projects/<NUM>/locations/global/workloadIdentityPools/github/providers/contoso
      service_account: sa-cicd@contoso-data-prod.iam.gserviceaccount.com
  - uses: hashicorp/setup-terraform@v3
  - run: terraform -chdir=infra apply -auto-approve
```

### 11.2 Org policies worth enforcing

| Constraint | Effect |
| --- | --- |
| `iam.disableServiceAccountKeyCreation` / `KeyUpload` | No SA keys |
| `iam.allowedPolicyMemberDomains` | Only your (and approved partner) domains in IAM |
| `gcp.resourceLocations` | Data only in approved regions (for example `in:eu-locations`) |
| `gcp.restrictCmekCryptoKeyProjects` / `gcp.restrictNonCmekServices` | Force CMEK for BigQuery |
| `bigquery.disableBQOmniAWS` / `Azure` | Block cross-cloud BigQuery Omni if not needed |
| `iam.automaticIamGrantsForDefaultServiceAccounts` | Stop default SAs from getting Editor |

### 11.3 VPC Service Controls

```bash
# Perimeter around data + API projects; BigQuery and Storage APIs restricted
gcloud access-context-manager perimeters create contoso_data \
  --policy=<ACCESS_POLICY_ID> --title="Contoso data" \
  --resources=projects/<DATA_PROJECT_NUM>,projects/<API_PROJECT_NUM> \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com,bigquerystorage.googleapis.com,analyticshub.googleapis.com
```

Effects:

- A valid identity outside the perimeter (for example a stolen token used from home) **can't** read BigQuery data in it.
- Queries inside the perimeter **can't** write results to a project outside it, which blocks `CREATE TABLE other_project.x AS SELECT ...` exfiltration.
- **Ingress rules** allow named external identities or sources in (for example `sa-northwind@...` from its client project for the direct-API pattern). **Egress rules** allow specific outbound flows (for example Analytics Hub subscriptions into client projects).
- Start in **dry-run mode** and watch *Policy Denied* logs for two weeks before enforcing.

### 11.4 Identity

- **Cloud Identity / Workspace** groups synced from your IdP. **Workforce Identity Federation** for external people, such as client analysts, who sign in with their own IdP.
- **2-step verification** enforced. Context-aware access (device, IP) through **Access Context Manager** access levels for console access.
- **Privileged Access Manager** for just-in-time elevation to `bigquery.admin` or `resourcemanager.projectIamAdmin`, with approval.
- **Quarterly reviews:** the IAM Recommender removes unused permissions, and Policy Analyzer answers "who can read `gold.sales`?".

### 11.5 Network and encryption

- **Private Google Access** / **Private Service Connect** for Google APIs, so on-premises and VPC workloads reach BigQuery without the public internet.
- **CMEK:** set a default KMS key on datasets (`default_kms_key_name`). Revoking the key makes data unreadable. **Cloud EKM** keeps keys outside Google.
- **Assured Workloads** (for example EU Sovereign Controls) for regulated data-residency and personnel-access requirements.
- **Data residency:** dataset location plus the `gcp.resourceLocations` org policy.

### 11.6 Lab

1. Move datasets, IAM and the row access policies into Terraform and deploy from GitHub Actions with WIF.
2. Convert `api.v_sales` into a Dataform model with assertions.
3. Create a VPC-SC perimeter in dry-run mode around `contoso-data-prod`, run the client API and Analytics Hub flows, and write the ingress and egress rules the logs show you need.
4. Enforce `iam.disableServiceAccountKeyCreation` and confirm `gcloud iam service-accounts keys create` fails.

## Module 12 — Capstone, quiz and cheat sheet

### 12.1 Capstone project

**Brief:** onboard three clients in one day. *Tailspin* wants a REST API with a key, *Woodgrove* uses BigQuery and wants live tables, and *Litware* needs a nightly bulk extract of 50 million rows into its own lake.

- [ ] Tailspin: Apigee developer app on the `Bronze` product; Cloud Run impersonates `sa-tailspin`; row access policy mapping added
- [ ] Woodgrove: per-client share dataset with authorized views, private Analytics Hub listing with egress controls
- [ ] Litware: Storage Read API extract as `sa-litware` (WIF from AWS), billed to its client project, capped by custom quota
- [ ] Policy tag + `EMAIL_MASK` on all PII; Sensitive Data Protection discovery active
- [ ] VPC-SC perimeter enforced, with ingress and egress rules for all three flows
- [ ] Everything in Terraform and Dataform, deployed keylessly from CI
- [ ] Cost and usage report by `client` label; budgets on each client project
- [ ] One-page data-sharing record per client

**Assessment:** the reviewer tries to read another client's rows through each channel, create a service-account key, copy the linked dataset out, and query from outside the perimeter with a valid token. All must fail.

### 12.2 Knowledge check

1. Which two permissions does any BigQuery query need, and on which resources?
2. Why put each client in its own job project?
3. What happens to users who are not grantees of any row access policy on a table?
4. What's the difference between denying a column with a policy tag and masking it with a data policy?
5. What does an authorized dataset give you over individual authorized views?
6. Why are Workload Identity Federation credential config files safe to hand to a client?
7. Who pays for queries on an Analytics Hub linked dataset?
8. Which API is best for a 50-million-row nightly extract, and does it respect row-level security?
9. Name two controls that stop a single on-demand query from costing a fortune.
10. What does VPC Service Controls protect against that IAM alone doesn't?

**Answers**

1. `bigquery.jobs.create` (for example `jobUser`) on the project running the job, and data read (for example `dataViewer`) on the dataset, table or view.
2. Its queries are billed and quota-limited separately, so cost attribution and caps are per client, while data stays in your project.
3. They see zero rows.
4. A policy tag without access gives *Access Denied*. A data policy with masked-reader access returns a masked value instead.
5. Every current and future view in the dataset is authorised automatically. There's less to maintain and less to forget.
6. They contain no secret. They only describe how to exchange the client's own IdP token, and your provider's attribute condition limits which identities are accepted.
7. The subscriber, in the project where the linked dataset lives.
8. The Storage Read API. Yes: row access policies, column-level security and masking all apply.
9. Any two of: `maximum_bytes_billed`; custom daily quotas per project or user; `require_partition_filter`; reservations with capped slots.
10. Exfiltration by a valid identity, for example a stolen token used outside your network, or copying query results into a project outside the perimeter.

### 12.3 Cheat sheet

| Need | Use |
| --- | --- |
| Technical client on GCP | Grant views; client queries via BigQuery API in its own job project |
| External system on AWS/Azure | Workload Identity Federation, no SA keys |
| Paid API product | Cloud Run + Apigee (or API Gateway); impersonate per-client SA |
| Client wants live tables | Analytics Hub listing (+ egress controls) |
| Aggregate-only sharing | Data clean room with aggregation threshold |
| Bulk extract | Storage Read API |
| Same KPIs everywhere | Looker / LookML (+ `access_filter`) |
| Hide rows | Row access policies (`SESSION_USER()` mapping) |
| Hide or mask columns | Policy tags + data policies (dynamic masking) |
| Expose without base-table access | Authorized views, datasets, routines |
| Find PII | Sensitive Data Protection discovery |
| Catalog, glossary, lineage | Dataplex Universal Catalog |
| Who did what | Cloud Audit Logs → BigQuery sink; `INFORMATION_SCHEMA.JOBS` |
| Data quality | Dataplex data quality and profile scans; Dataform assertions |
| Cost | Labels, billing export, budgets, custom quotas, reservations |
| Anti-exfiltration | VPC Service Controls |
| Safe change | Terraform + Dataform + keyless CI |

### Further reading

Google renames and regroups products often. Check names, editions and GA status before you deliver this course. The links below follow the usual Google Cloud docs paths but weren't individually checked for this draft.

- [BigQuery REST API: jobs.query](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/query)
- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Analytics Hub / BigQuery sharing](https://cloud.google.com/bigquery/docs/analytics-hub-introduction)
- [Storage Read API](https://cloud.google.com/bigquery/docs/reference/storage)
- [Row-level security](https://cloud.google.com/bigquery/docs/row-level-security-intro)
- [Column-level security and data masking](https://cloud.google.com/bigquery/docs/column-data-masking-intro)
- [Authorized views and datasets](https://cloud.google.com/bigquery/docs/authorized-views)
- [Dataplex Universal Catalog](https://cloud.google.com/dataplex/docs/catalog-overview)
- [VPC Service Controls for BigQuery](https://cloud.google.com/vpc-service-controls/docs/supported-products)
- [Apigee](https://cloud.google.com/apigee/docs)
