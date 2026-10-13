# Cloud Identity, Tokens & Secrets: Full Security Course

> Source of truth: the live Claude Doc (https://claude.ai/code/artifact/c21fe269-4c42-4ae0-82ac-ba47a9708299), which also holds the drawn diagrams. Exported 2026-10-13. Written from model knowledge; preview-era syntax is flagged in the text for verification.


Oct 13, 2026 · @Matt

## Course overview

Every cloud security setup — Azure, AWS, Google Cloud, Fabric, Databricks, Snowflake or GitHub — answers the same four questions: **who** is calling, **how** they prove it, **what** they may do, and **where** the secret proof is kept. Once you can map any product's vocabulary onto those four questions, managed identities, service principals, service accounts, OAuth, tokens and vaults stop being a jumble of names.

**Audience:** data engineers, analytics engineers, developers and team leads who connect systems together and want to do it securely without being security specialists.

**How each module works:** a plain-language explanation, a "why use it" section, the creation steps (portal and CLI), then **5 real examples**. Each example states the situation, why this identity or mechanism is the right choice, how to create it, and the code that uses it.

### The one mental model

&#91;embedded content: The four questions · identity, proof, permission, vault\]

The proof (highlighted) is where breaches happen, which is why every module pushes it towards options that store no secret at all.

| Question | Plain words | Azure / Entra | AWS | Google Cloud |
| --- | --- | --- | --- | --- |
| **Who?** (identity) | The account doing the work | User, group, service principal, managed identity | IAM user, IAM role | Google account, service account |
| **How proven?** (authentication) | The proof it is really them | Password+MFA, client secret, certificate, federated token | Password+MFA, access keys, STS temporary credentials | Password+2SV, SA key, attached SA metadata, federated token |
| **What allowed?** (authorization) | The permissions it has | Azure RBAC roles, Entra roles, app permissions (scopes) | IAM policies (JSON) | IAM roles bound on resources |
| **Where are secrets kept?** (vault) | Safe storage for passwords and keys | Azure Key Vault | Secrets Manager, KMS | Secret Manager, Cloud KMS |

### The golden rule of this course

> **The best secret is no secret.** Prefer, in this order: (1) a **managed identity / attached service account / IAM role**, where the cloud hands your code short-lived tokens automatically; (2) **workload identity federation**, where an outside system proves itself with its own token; (3) a **certificate**; (4) a **client secret or key stored in a vault** and rotated. A secret pasted into code, a notebook or a config file is never acceptable.

### Module map

| # | Module | You will be able to |
| --- | --- | --- |
| 1 | Core concepts | Explain identity, principal, authentication, authorization and least privilege |
| 2 | Microsoft Entra ID | Navigate tenants, users, groups, roles, Conditional Access and PIM |
| 3 | App registrations, Client ID, secrets | Register an app and choose between secret and certificate |
| 4 | Service principals | Create and scope an SP for automation |
| 5 | Managed identities | Remove secrets from Azure-hosted code |
| 6 | Google Cloud service accounts | Use attached SAs, impersonation and ADC without keys |
| 7 | AWS IAM | Use roles, instance profiles and AssumeRole instead of access keys |
| 8 | Workload identity federation | Connect clouds and CI/CD with no stored secret |
| 9 | Data-platform identities | Authenticate to Fabric, Databricks, Snowflake, BigQuery and GitHub |
| 10 | OAuth 2.0 and OIDC | Pick the right flow for every scenario |
| 11 | Tokens | Read a JWT and validate it; know API keys, PATs and SAS |
| 12 | Secret vaults | Store, reference and rotate secrets on every platform |
| 13 | Authorization: RBAC, RLS, OLS, least privilege | Design roles, row- and object-level security with least privilege on every platform |
| 14–15 | Comparison guide; best practices and quiz | Translate concepts across clouds, rotate, respond to leaks and test yourself |

Commands are written for Azure CLI, AWS CLI and gcloud as of late 2026. Portal labels move often, so trust the CLI over screenshots. Replace every `<placeholder>` with your own values, and never paste real secrets into documents like this one.

## Module 1 — Core concepts

Most security confusion comes from mixing up three ideas: the **identity** (the account), the **credential** (its proof) and the **permission** (what it may do). Keep them separate and every product makes sense.

### 1.1 Vocabulary

| Term | Meaning | Analogy |
| --- | --- | --- |
| **Identity** | An account that can be recognised: a person, an app, a VM, a pipeline | A named employee |
| **Principal** | Any identity that can be granted permissions (user, group, service principal, service account, role) | Anyone who can hold a key card |
| **Human identity** | Used by a person, interactively, with MFA | Staff member |
| **Workload (non-human) identity** | Used by software: service principal, managed identity, service account, IAM role | A delivery robot with its own badge |
| **Credential** | The proof: password, secret, certificate, private key, token | The badge itself |
| **Authentication (AuthN)** | Proving who you are. Fails with **HTTP 401 Unauthorized** | Showing your badge at the door |
| **Authorization (AuthZ)** | Deciding what you may do. Fails with **HTTP 403 Forbidden** | Which rooms your badge opens |
| **Tenant / directory** | The container of identities for one organisation (Entra tenant, Google Cloud org, AWS org/account) | The company's staff register |
| **Least privilege** | Give only the permissions needed, on the narrowest scope, for the shortest time | Badge opens one room, during working hours |
| **Token** | A short-lived, signed pass issued after authentication | A visitor sticker valid today only |
| **Secret** | Any value that grants access if stolen (password, client secret, API key, private key) | A master key |
| **Vault** | A hardened service that stores secrets, logs every read and supports rotation | The key safe at reception |

### 1.2 Why separate human and workload identities?

- People leave, change roles and go on holiday. A pipeline running under a person's account breaks when they leave, and the audit log blames the wrong actor.
- People can do MFA; software can't. Workload identities need different proof (certificates, managed tokens, federation).
- Permissions differ: a pipeline needs write access to one table, while its author may need much broader read access.

### 1.3 Five real examples

**Example 1 — Analyst opens a Power BI report (human, authN + authZ).**

- *Situation:* Sarah opens the Sales report.
- *AuthN:* Entra ID checks her password plus Authenticator push (MFA), then issues a token.
- *AuthZ:* Power BI checks whether Sarah, or a group she belongs to, has Viewer access to the workspace, and RLS restricts her rows.
- *Lesson:* signing in successfully says nothing about what she may see. Two separate checks happen.

**Example 2 — Nightly pipeline copies data (workload identity).**

- *Situation:* a Data Factory pipeline copies an SQL database into a Lakehouse at 02:00.
- *Wrong way:* storing an engineer's username and password in the linked service.
- *Right way:* the factory's **managed identity** gets `db_datareader` on the source and Contributor on the target workspace (Module 5).
- *Lesson:* software gets its own identity, with only the permissions it needs.

**Example 3 — Contractor needs read access for two weeks (least privilege + time limit).**

- *Situation:* an auditor must read Finance tables for 14 days.
- *Right way:* invite them as an Entra **B2B guest**, add them to `grp-finance-audit-read` with an **access package expiry** or **PIM for Groups** end date, and grant that group read-only access on one schema.
- *Lesson:* scope (one schema), level (read) and time (14 days) are all limited.

**Example 4 — Shared admin password (anti-pattern).**

- *Situation:* three engineers share the `sa` SQL login stored in a team chat.
- *Problems:* no MFA, no individual audit trail, impossible to revoke one person, and the secret is copied in chat history forever.
- *Fix:* named Entra accounts with Entra authentication on SQL, the `sa` password rotated into Key Vault for break-glass use only, and an alert on any `sa` login.

**Example 5 — 401 vs 403 when calling a partner API.**

- *401 Unauthorized:* your token is missing, expired, signed by the wrong tenant, or has the wrong audience. Fix the **authentication**: get a fresh token for the right resource.
- *403 Forbidden:* the token is valid, but the identity lacks the permission or role. Fix the **authorization**: grant the role or scope.
- *Lesson:* reading the status code tells you which half of the system to debug.

### 1.4 Check yourself

- [ ] I can name the identity, the credential and the permission in any integration I own.
- [ ] None of my automated jobs run as a human account.
- [ ] I know which of my secrets would cause damage if they were posted publicly today.

## Module 2 — Microsoft Entra ID

**Microsoft Entra ID** (called *Azure Active Directory* until 2023) is Microsoft's cloud identity provider. It holds the users, groups and app identities for Microsoft 365, Azure, Fabric, Power BI, Dynamics and thousands of SaaS apps, and it issues the OAuth tokens they all accept. *Entra* is also the product family name (Entra ID, Entra External ID, Entra Verified ID…). In everyday speech, "Entra" usually means Entra ID.

### 2.1 Building blocks

| Object | What it is | Key facts |
| --- | --- | --- |
| **Tenant** | Your organisation's dedicated Entra ID instance | Identified by a **Tenant ID** (GUID) and domains like `contoso.com`, `contoso.onmicrosoft.com` |
| **User** | Member (employee) or **guest** (B2B, from another tenant or email) | Guests sign in with their home credentials |
| **Group** | Security group or Microsoft 365 group; **dynamic** groups fill themselves from rules | Always grant access to groups, not people |
| **App registration** | The *definition* of an application (Module 3) | Has the **Client ID** |
| **Enterprise application / service principal** | The app's *identity* in a tenant (Module 4) | What you grant roles to |
| **Managed identity** | A service principal that Azure manages for you (Module 5) | No secret |
| **Entra roles** | Admin rights over **Entra itself** and M365 (Global Administrator, User Administrator, Application Administrator…) | Not the same as Azure RBAC |
| **Azure RBAC roles** | Rights over **Azure resources** (Owner, Contributor, Reader, Key Vault Secrets User…) at management-group, subscription, resource-group or resource scope | Not the same as Entra roles |
| **Conditional Access (CA)** | If/then policies: *if* user X from location Y on device Z wants app A, *then* require MFA, a compliant device, or block | Needs Entra ID P1 |
| **PIM** (Privileged Identity Management) | Just-in-time, time-limited, approved activation of admin roles | Needs Entra ID P2 / ID Governance |
| **Access reviews / entitlement management** | Periodic "does this person still need it?" and self-service access packages with expiry | ID Governance |

The classic confusion: **Global Administrator** (an Entra role) doesn't automatically give access to Azure subscriptions, and **Owner** of a subscription (an Azure RBAC role) can't reset user passwords. They're two separate permission systems sharing one directory.

### 2.2 Where to find the IDs you'll need

```bash
az login
az account show --query "{tenantId:tenantId, subscriptionId:id, user:user.name}"
az ad signed-in-user show --query "{objectId:id, upn:userPrincipalName}"
```

| ID | Looks like | Used for |
| --- | --- | --- |
| Tenant ID | `72f988bf-86f1-41af-91ab-2d7cd011db47` | Token endpoint `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token` |
| Object ID | GUID per user, group or SP | Role assignments, RLS mappings |
| Client ID (Application ID) | GUID per app registration | Identifies the app when requesting tokens |
| Subscription ID | GUID per Azure subscription | Azure resource scopes |

### 2.3 Five real examples

**Example 1 — Grant a data team access through a group.**

- *Why:* when people join or leave, you change group membership once instead of editing 20 resources.
- *How:*

```bash
az ad group create --display-name "grp-data-engineers" --mail-nickname grp-data-engineers
az ad group member add --group grp-data-engineers --member-id $(az ad user show --id sarah@contoso.com --query id -o tsv)
# Grant the group (not Sarah) Contributor on the data resource group
az role assignment create --assignee-object-id $(az ad group show -g grp-data-engineers --query id -o tsv) \
  --assignee-principal-type Group --role Contributor \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-data-prod
```

In Fabric, add the same group to the workspace as Contributor.

**Example 2 — Require MFA for everyone, and block legacy sign-in.**

- *Why:* passwords alone are phished every day. Legacy protocols (POP, IMAP, basic SMTP) can't do MFA.
- *How:* Entra admin center → **Protection → Conditional Access → New policy**:
  - Policy A: *All users* (exclude 2 break-glass accounts) → *All resources* → Grant: **Require multifactor authentication** (or *phishing-resistant MFA* for admins).
  - Policy B: *All users* → Conditions: *Client apps = Exchange ActiveSync + other clients* → **Block**.
- Start in **Report-only** mode for a week, then switch on.

**Example 3 — Just-in-time admin with PIM.**

- *Why:* standing admin rights are the top target for attackers. With PIM, nobody is an admin until they ask, with a reason, for a limited time.
- *How:* **ID Governance → Privileged Identity Management → Microsoft Entra roles → Global Administrator → Add assignment** → type **Eligible**. In role settings, require MFA plus justification plus approval, with a maximum of 4 hours. Do the same under **Azure resources** for subscription Owner, and under **Groups** for the Fabric Admin group.

**Example 4 — Invite a client analyst as a B2B guest.**

- *Why:* the client keeps using their own company account and MFA. When they leave their company, their access dies automatically.

```bash
az rest --method POST --url https://graph.microsoft.com/v1.0/invitations --body '{
  "invitedUserEmailAddress": "analyst@northwind.com",
  "inviteRedirectUrl": "https://app.powerbi.com",
  "sendInvitationMessage": true }'
```

Then add the guest to `grp-client-northwind`. Use **cross-tenant access settings** to trust Northwind's MFA claims, and an **access review** every 90 days.

**Example 5 — Dynamic group for all Finance staff.**

- *Why:* new Finance hires automatically get the Finance reports, and movers lose them, with no tickets.
- *How:* create a group with **Membership type: Dynamic User** and rule `(user.department -eq "Finance") and (user.accountEnabled -eq true)`. Grant the group Viewer on the Finance workspace. This relies on HR data populating `department` correctly through Entra Connect or HR provisioning.

### 2.4 Must-do settings for any tenant

- [ ] Two **break-glass** cloud-only Global Admin accounts, with FIDO2 keys stored in a safe, excluded from CA, and monitored.
- [ ] No permanent Global Admins apart from break-glass; everyone else through PIM.
- [ ] MFA for all users through Conditional Access; phishing-resistant MFA for admins.
- [ ] *Users can register applications* = **No** (Module 3), and user consent restricted to verified publishers.
- [ ] Sign-in and audit logs exported to Log Analytics or Sentinel (default retention is short).

## Module 3 — App registrations, Client ID, Tenant ID, secrets and certificates

An **app registration** is how you tell Entra ID that a piece of software exists. It gives the software a **Client ID**, lets you attach credentials (a **client secret**, a **certificate** or a **federated credential**), and declares which permissions it needs. A Client ID on its own is *not* secret: it's a username. The secret or certificate is the password.

### 3.1 App registration vs enterprise application

|  | App registration | Enterprise application (service principal) |
| --- | --- | --- |
| What | The global **definition** (blueprint) of the app | The app's **instance/identity** inside one tenant |
| How many | One, in the home tenant | One per tenant where the app is used |
| Holds | Client ID, credentials, redirect URIs, requested permissions, exposed scopes, app roles | Role assignments, consent grants, user assignment, sign-in logs |
| Analogy | A company's job description for "Courier" | The actual courier badge issued at each building |

Creating an app registration in the portal automatically creates its service principal in your tenant. That's why both appear (Module 4).

### 3.2 The values you'll copy

| Value | Secret? | Example | Where |
| --- | --- | --- | --- |
| **Tenant ID** (Directory ID) | No | `72f988bf-...` | App → Overview |
| **Client ID** (Application ID) | No | `0d3c1a2b-...` | App → Overview |
| **Object ID** of the app registration | No | GUID | App → Overview (not used for tokens) |
| **Object ID** of the service principal | No | GUID | Enterprise apps → the app (used for RBAC and RLS) |
| **Client secret value** | **Yes** | `Abc8Q~...` | Shown **once** when created; copy straight into a vault |
| **Certificate private key** | **Yes** | `.pfx` / `.pem` | Stays with you (or in Key Vault); only the public key is uploaded |

### 3.3 Secret vs certificate vs federated credential

| Credential | How it proves identity | Lifetime | Risk if leaked | Use when |
| --- | --- | --- | --- | --- |
| **Client secret** | Sends the shared string to Entra | Max 24 months (aim for ≤ 6) | Anyone with the string becomes the app | Quick tests, tools that support nothing else |
| **Certificate** | Signs a short JWT ("client assertion") with a private key; Entra checks the signature with the uploaded public key | 1–2 years typical | Private key must be stolen, not just a string; can be non-exportable in Key Vault or an HSM | Production apps outside Azure |
| **Federated credential** | Presents a token from a trusted outside issuer (GitHub, GCP, AWS, Kubernetes) | No stored credential | Nothing to leak | CI/CD and cross-cloud (Module 8) |
| **Managed identity** | Azure fetches tokens for you | No stored credential | Nothing to leak | Code running in Azure (Module 5) |

### 3.4 Permissions: delegated vs application

- **Delegated permissions (scopes)**: the app acts *on behalf of a signed-in user* and can never do more than that user could. Example: `User.Read`, `Files.Read`.
- **Application permissions (app roles)**: the app acts *as itself*, with no user. It's powerful and needs **admin consent**. Example: `User.Read.All` lets the app read every user in the tenant.
- **Admin consent** is a tenant admin approving the permissions for the whole organisation.
- Ask for the least you need. `Sites.Selected` (grant to specific SharePoint sites) beats `Sites.Read.All`.

### 3.5 Five real examples

**Example 1 — Script that lists users through Microsoft Graph (client secret, app-only).**

- *Why:* a nightly HR reconciliation job runs unattended outside Azure, so it needs an app identity.
- *Create:*

```bash
az ad app create --display-name "app-hr-recon" --sign-in-audience AzureADMyOrg
APP_ID=$(az ad app list --display-name app-hr-recon --query "[0].appId" -o tsv)
az ad sp create --id $APP_ID
# Add application permission User.Read.All (Graph role id df021288-bdef-4463-88db-98f22de89214)
az ad app permission add --id $APP_ID --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions df021288-bdef-4463-88db-98f22de89214=Role
az ad app permission admin-consent --id $APP_ID
# Secret valid 180 days -> straight into Key Vault, never to the screen in real life
az ad app credential reset --id $APP_ID --display-name nightly --years 0.5 --query password -o tsv \
  | xargs -I{} az keyvault secret set --vault-name kv-contoso --name hr-recon-secret --value {}
```

- *Use:*

```python
import msal, requests
app = msal.ConfidentialClientApplication(
    client_id="<APP_ID>", authority="https://login.microsoftonline.com/<TENANT_ID>",
    client_credential="<secret read from Key Vault at runtime>")
tok = app.acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])
users = requests.get("https://graph.microsoft.com/v1.0/users?$select=displayName,mail",
                     headers={"Authorization": f"Bearer {tok['access_token']}"}).json()
```

`/.default` means "all the application permissions already consented for this resource".

**Example 2 — Same job, with a certificate instead of a secret.**

- *Why:* production. The private key can live non-exportable in Key Vault, and nothing in config files grants access by itself.

```bash
az keyvault certificate create --vault-name kv-contoso -n app-hr-recon-cert \
  -p "$(az keyvault certificate get-default-policy)"
az ad app credential reset --id $APP_ID --keyvault kv-contoso --cert app-hr-recon-cert --append
```

```python
app = msal.ConfidentialClientApplication(
    client_id="<APP_ID>", authority="https://login.microsoftonline.com/<TENANT_ID>",
    client_credential={"private_key": open("key.pem").read(), "thumbprint": "<CERT_THUMBPRINT>"})
```

MSAL builds and signs the client assertion for you.

**Example 3 — Internal web app with "Sign in with Microsoft" (delegated).**

- *Why:* staff should log in with their company account and MFA, and the app reads only their own profile.
- *Create:* **App registrations → New** → *Accounts in this organizational directory only* → Redirect URI (Web) `https://intranet.contoso.com/auth/callback`. Permissions: delegated `openid`, `profile`, `User.Read`. Create a certificate or secret for the server side.
- *Flow:* authorization code + PKCE (Module 10). The app receives an **ID token** (who the user is) and an **access token** for Graph `User.Read`.

**Example 4 — Protect your own FastAPI with Entra (expose an API).**

- *Why:* clients and internal apps call your API with Entra tokens, so you don't invent your own auth.
- *Create:* register `api-sales`. Under **Expose an API**, set Application ID URI `api://api-sales` and add scope `Sales.Read` (delegated). Under **App roles**, add `Sales.Read.All` (allowed member type: Applications) for app-only callers.
- Grant a caller app the role: caller's app registration → *API permissions* → *My APIs* → `api-sales` → Application → `Sales.Read.All` → admin consent.
- The caller requests scope `api://api-sales/.default`, and your API validates the token (Module 11, example 2) and checks `roles` contains `Sales.Read.All`.

**Example 5 — Multi-tenant SaaS used by customers in their own tenants.**

- *Why:* your product must read each customer's data in their tenant, with their admin's consent.
- *Create:* set *Supported account types* to **Accounts in any organizational directory (multi-tenant)**. Each customer admin visits `https://login.microsoftonline.com/<customer-tenant>/adminconsent?client_id=<APP_ID>`, which creates a service principal in **their** tenant. You request tokens with `authority=https://login.microsoftonline.com/<customer-tenant>`.
- *Security:* publisher verification, minimum permissions, and per-tenant token caching. Never use a single-tenant token for another tenant.

### 3.6 Hygiene checklist

- [ ] Tenant setting *Users can register applications* = No; registrations via a request process.
- [ ] Every app registration has **two owners** (people or groups) and a description.
- [ ] No secrets older than 6 months; prefer certificates or federation; expiry alerts in place.
- [ ] Review application permissions quarterly (`*.ReadWrite.All` is a red flag).

## Module 4 — Service principals (Azure / Entra)

A **service principal (SP)** is the identity of an application inside a tenant: the thing you grant roles to. Users have user accounts; software has service principals. Every app registration has one, every managed identity *is* one, and Microsoft's own apps (Power BI, Graph, Fabric) have one in your tenant too.

### 4.1 Three kinds of service principal

| Kind | Created by | Credential | Example |
| --- | --- | --- | --- |
| **Application** | Your app registration (or a multi-tenant app's consent) | Secret, certificate or federated credential *you* manage | `sp-terraform-prod`, `sp-client-northwind` |
| **Managed identity** | Azure, when you enable it on a resource or create a user-assigned identity | None you can see; Azure rotates it | Data Factory's identity, `id-api-prod` |
| **Legacy / first-party** | Microsoft | n/a | "Power BI Service", "Microsoft Graph" |

### 4.2 Control plane vs data plane: the RBAC trap

Azure separates **managing** a resource from **using its data**:

| You want to | Control-plane role (not enough!) | Data-plane role you actually need |
| --- | --- | --- |
| Read blobs in ADLS Gen2 | Reader / Contributor on the storage account | **Storage Blob Data Reader / Contributor** |
| Read Key Vault secrets | Key Vault Contributor | **Key Vault Secrets User** (RBAC mode) |
| Query Azure SQL | SQL DB Contributor | **A database user** `CREATE USER [sp] FROM EXTERNAL PROVIDER` + `db_datareader` |
| Send to Event Hubs | Contributor | **Azure Event Hubs Data Sender** |
| Use a Fabric item | (none in Azure RBAC) | **Fabric workspace role** or item permission |

Contributor on a storage account can regenerate its keys, which is effectively full access. That's one more reason to disable shared-key access (Module 12).

### 4.3 Creating a service principal

```bash
# One command: app registration + service principal + secret + role assignment
az ad sp create-for-rbac --name "sp-terraform-prod" \
  --role Contributor --scopes /subscriptions/<sub-id>/resourceGroups/rg-data-prod \
  --years 0.5
# -> appId (client id), password (secret, shown once), tenant
```

The standard environment variables recognised by the Azure SDKs, Terraform and the CLI:

```bash
export AZURE_TENANT_ID=<tenant-id>
export AZURE_CLIENT_ID=<app-id>
export AZURE_CLIENT_SECRET=<from-vault>        # or AZURE_CLIENT_CERTIFICATE_PATH
```

```python
from azure.identity import DefaultAzureCredential
cred = DefaultAzureCredential()   # tries env vars -> workload identity -> managed identity -> az login ...
```

`DefaultAzureCredential` is why the same code runs on your laptop (with `az login`), in CI (with env vars or federation) and in Azure (with a managed identity) without changes.

### 4.4 Five real examples

**Example 1 — Terraform deploying infrastructure.**

- *Why:* infrastructure changes must come from a pipeline identity, not a person.
- *Create:* `sp-terraform-prod` with **Contributor** on the target resource groups only (plus **User Access Administrator** with a condition, *only if* Terraform must create role assignments).
- *Better:* give it a **federated credential** for GitHub Actions (Module 8), so no secret exists.
- *Use:* `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`, `ARM_USE_OIDC=true` in the azurerm provider.

**Example 2 — Calling the Fabric / Power BI REST API from a script.**

- *Why:* refresh semantic models, deploy pipelines or export reports automatically.
- *Create:*
  1. App registration `sp-fabric-automation` with a certificate.
  2. Add its SP to the Entra group `sg-fabric-api-clients`.
  3. Fabric Admin portal → Tenant settings → *Service principals can use Fabric APIs* → enabled for that group.
  4. Add the SP to the workspace as **Contributor** (or Member).
- *Use:*

```python
from azure.identity import CertificateCredential
import requests
cred = CertificateCredential("<tenant>", "<app-id>", certificate_path="sp-fabric.pem")
tok = cred.get_token("https://api.fabric.microsoft.com/.default").token
requests.post(f"https://api.fabric.microsoft.com/v1/workspaces/<ws>/items/<model-id>/jobs/instances?jobType=Refresh",
              headers={"Authorization": f"Bearer {tok}"})
```

**Example 3 — Databricks (or any Spark job outside Azure) reading ADLS Gen2.**

- *Why:* the job needs to read the `raw` container and nothing else.
- *Create:* SP `sp-dbx-raw-reader` → role **Storage Blob Data Reader** on the `raw` **container** scope (not the account):

```bash
az role assignment create --assignee <app-id> --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub>/resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/contosolake/blobServices/default/containers/raw"
```

- On Databricks with Unity Catalog, prefer an **access connector** (a managed identity) in a storage credential, so no SP secret is needed at all.

**Example 4 — An app connecting to Azure SQL Database without SQL passwords.**

- *Why:* Entra authentication gives MFA for humans, central disabling and no SQL logins to rotate.
- *Create:* in the database, as an Entra admin:

```sql
CREATE USER [sp-sales-api] FROM EXTERNAL PROVIDER;   -- display name of the SP
ALTER ROLE db_datareader ADD MEMBER [sp-sales-api];
```

- *Use:* ODBC Driver 18 with `Authentication=ActiveDirectoryServicePrincipal;UID=<app-id>;PWD=<secret>`, or better `ActiveDirectoryMsi` if the app runs in Azure (Module 5).

**Example 5 — One service principal per external client.**

- *Why:* each client is separately auditable and revocable, and data-level security can map the SP to that client (Fabric RLS, SQL RLS).
- *Create:* `sp-client-northwind`, `sp-client-fabrikam`… each with a certificate (or a federated credential trusting the client's own IdP). Add tags *owner* and *contract end date*, and grant only `SELECT` on the API schema.
- *Operate:* monitor **Entra → Sign-in logs → Service principal sign-ins**. To offboard, disable the SP (`az ad sp update --id <id> --set accountEnabled=false`) and delete its credentials.

### 4.5 Hygiene checklist

- [ ] One SP per purpose. Never reuse one SP across unrelated systems.
- [ ] Scope roles to the resource group or resource, never the whole subscription without a reason.
- [ ] No SP holds *Owner* or *Global Administrator*, except a break-glass automation SP that's documented and monitored.
- [ ] Workload identity sign-ins reviewed; stale SPs (no sign-in for 90 days) disabled. **Entra Workload ID** premium adds Conditional Access and risk detection for SPs.

## Module 5 — Managed identities

A **managed identity (MI)** is a service principal that Azure creates and looks after for you. Code running on an Azure resource (VM, App Service, Function, Container App, AKS pod, Data Factory, Synapse, Logic App, Databricks access connector, Automation…) asks a local endpoint for a token, and Azure answers. There's **no secret to create, store, rotate or leak**. Whenever the code runs in Azure, this is the first choice.

### 5.1 System-assigned vs user-assigned

|  | System-assigned | User-assigned |
| --- | --- | --- |
| Lifecycle | Born and deleted with **one** resource | Standalone Azure resource (`Microsoft.ManagedIdentity/userAssignedIdentities`) |
| Sharing | One resource only | Attach to **many** resources |
| Pre-provisioning | Role grants only after the resource exists | Create identity + grants first, then deploy resources (good for IaC) |
| Typical use | A single Function App, a Data Factory | A fleet of container replicas, blue/green slots, a Databricks access connector reused across workspaces |
| Name in Entra | Same as the resource | Whatever you call it (`id-sales-api-prod`) |

### 5.2 How it works under the hood

1. Your code (through `azure-identity`) calls the local **Instance Metadata Service** (`http://169.254.169.254/metadata/identity/oauth2/token`) or, on App Service and Functions, the `IDENTITY_ENDPOINT`.
2. Azure returns an access token for the resource you asked for (`https://storage.azure.com/`, `https://vault.azure.net`, `https://database.windows.net/`…).
3. Your code calls the target service with `Authorization: Bearer <token>`. The service checks the MI's role assignment.

The endpoint is reachable only from inside that resource, so nobody outside can impersonate it.

### 5.3 Five real examples

**Example 1 — Azure Function reads secrets from Key Vault.**

- *Why:* the function needs a third-party API key. Storing it in app settings exposes it to everyone with portal access.
- *Create:*

```bash
az functionapp identity assign -g rg-app -n func-orders
PRINCIPAL=$(az functionapp identity show -g rg-app -n func-orders --query principalId -o tsv)
az role assignment create --assignee-object-id $PRINCIPAL --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" --scope $(az keyvault show -n kv-contoso --query id -o tsv)
```

- *Use, either way:*
  - App setting reference (no code): `PARTNER_API_KEY=@Microsoft.KeyVault(SecretUri=https://kv-contoso.vault.azure.net/secrets/partner-api-key/)`
  - In code:

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
key = SecretClient("https://kv-contoso.vault.azure.net", DefaultAzureCredential()).get_secret("partner-api-key").value
```

**Example 2 — Data Factory copies from Azure SQL to ADLS.**

- *Why:* no SQL password in a linked service, and no storage key.
- *Create:* the factory's system-assigned MI exists by default.
  - SQL: `CREATE USER [adf-contoso-prod] FROM EXTERNAL PROVIDER; ALTER ROLE db_datareader ADD MEMBER [adf-contoso-prod];`
  - Storage: role **Storage Blob Data Contributor** on the `landing` container.
- *Use:* in each linked service choose **Authentication type: System-assigned managed identity**.

**Example 3 — Container App (FastAPI) queries a Fabric Warehouse.**

- *Why:* this is the client API from the data-APIs course. There's no secret in the container.
- *Create:* user-assigned `id-sales-api-prod` (so new revisions keep the same identity). Add it to `sg-fabric-api-clients` (tenant setting), then share the Warehouse with it (Read) and `GRANT SELECT ON SCHEMA::api TO [id-sales-api-prod]`.
- *Use:* ODBC connection string `Authentication=ActiveDirectoryMsi;UID=<client-id-of-the-UAMI>;`, or `DefaultAzureCredential(managed_identity_client_id="<client-id>")` for REST calls.

**Example 4 — VM-hosted legacy app uploads files to Blob Storage.**

- *Why:* the old app used a storage account key in a config file. That one key gives full control of the account.
- *Create:* enable system-assigned MI on the VM, grant **Storage Blob Data Contributor** on the `exports` container, then **disable shared key access** on the storage account.
- *Use:* `azcopy login --identity` then `azcopy copy ./out https://contosolake.blob.core.windows.net/exports/ --recursive`. Or use `BlobServiceClient(url, credential=DefaultAzureCredential())` in code.

**Example 5 — Databricks Unity Catalog reads ADLS through an access connector.**

- *Why:* Unity Catalog needs one trusted identity to reach storage on behalf of all governed users. A managed identity removes SP secrets from Databricks entirely.
- *Create:* deploy **Access Connector for Azure Databricks** (it has a system- or user-assigned MI). Grant **Storage Blob Data Contributor** on the lake. In Databricks: `CREATE STORAGE CREDENTIAL sc_lake WITH (AZURE_MANAGED_IDENTITY (ACCESS_CONNECTOR_ID = '<resource-id>'))`, then external locations on top of it.

### 5.4 Limits and gotchas

- Managed identities work only for code **running in Azure**. For GitHub, AWS, GCP, on-premises or a laptop, use workload identity federation (Module 8) or, as a last resort, a certificate.
- A **user-assigned** MI on a resource with several identities must be selected explicitly (`managed_identity_client_id`).
- Role assignments can take a few minutes to propagate, and tokens are cached for up to 24 hours, so revoking a role isn't instant for already-issued tokens.
- Anyone who can **run code** on the resource (deploy to the Function, SSH to the VM) can use its identity. Protect deployment rights as carefully as the identity's roles.

## Module 6 — Google Cloud service accounts

A **service account (SA)** is Google Cloud's workload identity. It covers both what Azure calls a service principal and a managed identity. It has an email address (`name@project.iam.gserviceaccount.com`), it's granted IAM roles like any principal, and code gets its tokens automatically when the SA is **attached** to the resource it runs on. Google's equivalent of the client secret is the **service-account key** (a JSON file containing a private key), and the guidance is simple: **don't create keys**.

### 6.1 Kinds of service account

| Kind | Example | Notes |
| --- | --- | --- |
| **User-managed** | `sa-sales-api@contoso-api-prod.iam.gserviceaccount.com` | You create it; one per workload |
| **Default** | `<project-number>-compute@developer.gserviceaccount.com` | Auto-created; historically got **Editor**, which is far too broad. Don't use it; disable automatic grants by org policy |
| **Service agents** | `service-<num>@gcp-sa-bigquery...` | Google-managed, used by Google services acting in your project. Leave them alone |

### 6.2 Two permissions people mix up

| Role | Granted **on** | Means |
| --- | --- | --- |
| `roles/bigquery.dataViewer` (any role) | A resource (dataset, bucket, project) | The SA **can do** that on the resource |
| `roles/iam.serviceAccountUser` | The **SA itself** | A person can **attach** ("actAs") this SA to a VM, Cloud Run or a job |
| `roles/iam.serviceAccountTokenCreator` | The **SA itself** | A person or another SA can **impersonate** it (mint its tokens) |

Whoever can attach or impersonate an SA effectively *has* its permissions. Grant these two roles as carefully as the SA's own roles.

### 6.3 Application Default Credentials (ADC)

Google client libraries find credentials in this order:

1. `GOOGLE_APPLICATION_CREDENTIALS` env var → a key file **or** a workload-identity-federation config (Module 8).
2. User credentials from `gcloud auth application-default login` (your laptop), optionally impersonating an SA.
3. The **attached SA** through the metadata server (Cloud Run, GCE, GKE, Functions, Dataflow…).

The same code therefore runs everywhere: `bigquery.Client()` needs no credentials argument.

### 6.4 Five real examples

**Example 1 — Cloud Run service reads BigQuery (attached SA).**

- *Why:* the API needs to query one dataset. With an attached SA there's no key, and tokens are rotated by Google.

```bash
gcloud iam service-accounts create sa-sales-api --display-name="Sales API"
SA=sa-sales-api@contoso-api-prod.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding contoso-api-prod --member=serviceAccount:$SA --role=roles/bigquery.jobUser
bq add-iam-policy-binding --member=serviceAccount:$SA --role=roles/bigquery.dataViewer contoso-data-prod:api
gcloud run deploy sales-api --image=<image> --service-account=$SA --region=europe-west1
```

```python
from google.cloud import bigquery
rows = bigquery.Client(project="contoso-api-prod").query("SELECT 1").result()   # uses the attached SA
```

**Example 2 — Developer tests as the SA from a laptop (impersonation, no key).**

- *Why:* engineers need to reproduce "works in Cloud Run, fails locally" without downloading a key.

```bash
# grant the engineer the right to mint tokens for the SA
gcloud iam service-accounts add-iam-policy-binding $SA \
  --member=user:sarah@contoso.com --role=roles/iam.serviceAccountTokenCreator
# then, on the laptop
gcloud auth application-default login --impersonate-service-account=$SA
bq query --use_legacy_sql=false --impersonate_service_account=$SA 'SELECT COUNT(*) FROM `contoso-data-prod.api.v_sales`'
```

Every call is logged with both identities: Sarah acting as the SA.

**Example 3 — Cloud Scheduler triggers a private Cloud Run job (OIDC token).**

- *Why:* the job endpoint must not be public, but the scheduler needs to call it every night.

```bash
gcloud iam service-accounts create sa-scheduler
gcloud run services add-iam-policy-binding nightly-export --region=europe-west1 \
  --member=serviceAccount:sa-scheduler@contoso-api-prod.iam.gserviceaccount.com --role=roles/run.invoker
gcloud scheduler jobs create http nightly-export --schedule="0 2 * * *" \
  --uri=https://nightly-export-xxxx.a.run.app/run --http-method=POST \
  --oidc-service-account-email=sa-scheduler@contoso-api-prod.iam.gserviceaccount.com \
  --oidc-token-audience=https://nightly-export-xxxx.a.run.app
```

The scheduler attaches a Google-signed **ID token** for that SA, and Cloud Run verifies it before your code runs.

**Example 4 — GKE pods get their own identity (Workload Identity Federation for GKE).**

- *Why:* without it, every pod on a node shares the node's SA. With it, each Kubernetes service account maps to its own IAM permissions.

```bash
# Cluster created with --workload-pool=contoso-api-prod.svc.id.goog
# Grant directly to the Kubernetes service account principal:
gcloud projects add-iam-policy-binding contoso-api-prod --role=roles/bigquery.jobUser \
  --member="principal://iam.googleapis.com/projects/<NUM>/locations/global/workloadIdentityPools/contoso-api-prod.svc.id.goog/subject/ns/sales/sa/sales-api"
```

Pods running as KSA `sales/sales-api` now get tokens with exactly that access.

**Example 5 — A SaaS vendor (Fivetran, dbt Cloud, Looker) needs BigQuery access.**

- *Why:* the old habit was "create an SA key and upload it to the vendor". The vendor then holds a permanent key to your project.
- *Better:* most vendors run as **their own** SA and show it in their setup screen. Grant **that** email the narrow roles (for example `bigquery.dataEditor` on one dataset + `jobUser`). There's no key, and access is revocable by deleting the binding.
- *If a key is truly unavoidable:* create it on a dedicated SA with minimal roles, set an expiry with the org policy `iam.serviceAccountKeyExpiryHours`, store it in Secret Manager, alert on key use, and rotate quarterly.

### 6.5 Hygiene checklist

- [ ] Org policy `iam.disableServiceAccountKeyCreation` enforced (exceptions per project, documented).
- [ ] Org policy `iam.automaticIamGrantsForDefaultServiceAccounts` enforced; default compute SA unused.
- [ ] One SA per workload; no SA with `Owner` or `Editor`.
- [ ] Review `serviceAccountUser` and `serviceAccountTokenCreator` grants; IAM Recommender for unused permissions; **unused SAs disabled**.

## Module 7 — AWS IAM: users, roles, instance profiles, STS

In AWS the key idea is the **IAM role**: an identity with permissions but **no long-term credentials**. Someone or something *assumes* the role and gets **temporary credentials** from STS (Security Token Service) that last 15 minutes to 12 hours. AWS roles are the equivalent of Azure managed identities and service principals, and of Google service accounts. **IAM users with access keys** are the equivalent of client secrets and SA keys, and should be rare.

### 7.1 Building blocks

| Object | What it is | Use it for |
| --- | --- | --- |
| **Root user** | The account's email login with unlimited power | Nothing day to day. MFA, lock away, alarm on use |
| **IAM user** | Long-lived identity, password and/or **access keys** (`AKIA...` + secret) | Almost never. Legacy tools only |
| **IAM role** | Identity with a **trust policy** (who may assume it) and **permission policies** (what it may do) | All workloads, cross-account access, federated humans |
| **Instance profile** | Wrapper that attaches a role to an EC2 instance | EC2 workloads |
| **Task role / execution role** | Role attached to an ECS task or Lambda function | Containers, serverless |
| **IAM Identity Center** (formerly AWS SSO) | Workforce SSO with your IdP (Entra, Okta), permission sets mapped to roles in each account | All humans |
| **Policy** | JSON document: `Effect`, `Action`, `Resource`, `Condition` | Permissions, trust, resource policies |
| **SCP / RCP** | Organization guardrails that cap what any identity in an account can do | Region locks, deny disabling CloudTrail |
| **Permission boundary** | Maximum permissions a role can ever have | Letting teams create roles safely |

### 7.2 Anatomy of a role

Trust policy (*who can become this role*):

```json
{ "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole" }] }
```

Permission policy (*what the role can do*):

```json
{ "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::contoso-landing/orders/*" },
    { "Effect": "Allow", "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:eu-west-1:111122223333:secret:partner-api-key-*" } ] }
```

The AWS SDK **default credential chain** looks in this order: env vars → shared config/SSO profile → web identity token (federation) → container credentials (ECS) → instance profile (EC2). So `boto3.client("s3")` needs no keys in code.

### 7.3 Five real examples

**Example 1 — Lambda reads S3 and a secret (execution role).**

- *Why:* the function runs inside AWS, so the role delivers temporary credentials automatically.

```bash
aws iam create-role --role-name lambda-orders-role --assume-role-policy-document file://trust-lambda.json
aws iam put-role-policy --role-name lambda-orders-role --policy-name orders-access --policy-document file://perm.json
aws iam attach-role-policy --role-name lambda-orders-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws lambda create-function --function-name orders --role arn:aws:iam::111122223333:role/lambda-orders-role ...
```

```python
import boto3
key = boto3.client("secretsmanager").get_secret_value(SecretId="partner-api-key")["SecretString"]
```

**Example 2 — EC2 app writes to S3 (instance profile instead of access keys).**

- *Why:* access keys in `~/.aws/credentials` on a server get copied into images and backups and leak.

```bash
aws iam create-instance-profile --instance-profile-name ec2-exports
aws iam add-role-to-instance-profile --instance-profile-name ec2-exports --role-name ec2-exports-role
aws ec2 associate-iam-instance-profile --instance-id i-0abc --iam-instance-profile Name=ec2-exports
```

Require **IMDSv2** on the instance (`HttpTokens=required`) so SSRF bugs can't steal the role's credentials from the metadata service.

**Example 3 — Containerised API on ECS Fargate (task role).**

- *Why:* each service gets its own permissions, even when several share a cluster.
- *Create:* the role trusts `ecs-tasks.amazonaws.com` and grants, for example, `dynamodb:GetItem` on one table. In the task definition, set `"taskRoleArn": "arn:aws:iam::111122223333:role/sales-api-task"`. (The separate *execution role* is for ECS itself, to pull images and read secrets injected as env vars.)

**Example 4 — Engineers sign in with Entra ID through IAM Identity Center.**

- *Why:* no IAM users and no long-lived keys on laptops. Leavers lose AWS access when they're disabled in Entra.
- *Create:* IAM Identity Center → external IdP = Entra (SAML + SCIM). Create **permission sets** (`DataEngineer`, `ReadOnly`) and assign Entra groups to accounts with them.
- *Use:*

```bash
aws configure sso          # once: start URL, region, account, permission set -> profile "data-prod"
aws sso login --profile data-prod
aws s3 ls --profile data-prod     # temporary credentials, expire after the session duration
```

**Example 5 — Snowflake (or any SaaS) reads your S3 bucket: cross-account role with External ID.**

- *Why:* the vendor's AWS account needs access, but giving them keys is risky. The **External ID** prevents the *confused deputy* problem, where another customer of the same vendor tricks it into using your role.
- *Create:* role `snowflake-s3-read` with a permission policy of `s3:GetObject` and `s3:ListBucket` on one bucket and prefix, and this trust policy:

```json
{ "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow",
    "Principal": { "AWS": "<STORAGE_AWS_IAM_USER_ARN from DESC INTEGRATION>" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>" } } }] }
```

```sql
CREATE STORAGE INTEGRATION s3_landing TYPE = EXTERNAL_STAGE STORAGE_PROVIDER = 'S3' ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::111122223333:role/snowflake-s3-read'
  STORAGE_ALLOWED_LOCATIONS = ('s3://contoso-landing/snowflake/');
DESC INTEGRATION s3_landing;   -- copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID into the trust policy
```

### 7.4 Hygiene checklist

- [ ] Root: MFA (hardware key), no access keys, CloudTrail alarm on root sign-in.
- [ ] No IAM users for humans; IAM Identity Center only.
- [ ] No access keys older than 90 days, ideally none at all (IAM credential report).
- [ ] SCPs deny leaving the organisation, disabling CloudTrail and GuardDuty, and unapproved regions.
- [ ] **IAM Access Analyzer** for public and cross-account findings and unused-access findings.
- [ ] IMDSv2 required on all EC2.

## Module 8 — Workload identity federation: no secrets across clouds and CI/CD

**Workload identity federation (WIF)** lets a workload that already has an identity *somewhere* (a GitHub Actions run, a GCP service account, an AWS role, a Kubernetes pod) prove itself to *another* cloud by presenting a signed token from its home issuer. The target cloud verifies the signature and the claims, then hands out its own short-lived token. Nothing is stored, so nothing can leak. This is how you remove the last client secrets and SA keys.

### 8.1 How the exchange works

1. The workload gets an **OIDC token** from its home issuer, for example `https://token.actions.githubusercontent.com`. Its claims say who it is: `sub = repo:contoso/data-platform:ref:refs/heads/main`.
2. It sends that token to the target's token service: Entra `/oauth2/v2.0/token` with `client_assertion`, AWS `sts:AssumeRoleWithWebIdentity`, or Google `sts.googleapis.com`.
3. The target checks the **issuer**, the **audience** and the **subject** (and any extra conditions) against a configured trust.
4. If they match, it returns a short-lived access token for the mapped identity.

| Target | The trust is configured on | Called |
| --- | --- | --- |
| **Microsoft Entra** | An app registration or user-assigned managed identity | **Federated identity credential** (issuer + subject + audience) |
| **AWS** | An IAM OIDC identity provider + role trust policy | `AssumeRoleWithWebIdentity` with `Condition` on `sub` / `aud` |
| **Google Cloud** | A workload identity pool + provider | **Workload Identity Federation**, with attribute mapping and attribute conditions |
| **Databricks / Snowflake** | Federation policy / `WORKLOAD_IDENTITY` on a service user | OAuth token federation / workload identity federation |

**Golden rule:** make the subject match as **specific** as possible. Trusting `repo:contoso/*` lets every repo, and every branch including forks' pull requests if misconfigured, act as your production identity. Pin repo + branch or environment.

### 8.2 Five real examples

**Example 1 — GitHub Actions deploys to Azure (no client secret).**

```bash
az identity create -g rg-cicd -n id-gh-deploy
az identity federated-credential create -g rg-cicd --identity-name id-gh-deploy -n gh-main \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:contoso/data-platform:environment:production" \
  --audiences api://AzureADTokenExchange
az role assignment create --assignee $(az identity show -g rg-cicd -n id-gh-deploy --query principalId -o tsv) \
  --role Contributor --scope /subscriptions/<sub>/resourceGroups/rg-data-prod
```

```yaml
jobs:
  deploy:
    environment: production            # must match the subject above
    permissions: { id-token: write, contents: read }
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: <client-id of id-gh-deploy>
          tenant-id: <tenant-id>
          subscription-id: <sub-id>
      - run: az deployment group create -g rg-data-prod -f main.bicep
```

**Example 2 — GitHub Actions deploys to AWS.**

```bash
aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

```json
{ "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                      "token.actions.githubusercontent.com:sub": "repo:contoso/data-platform:ref:refs/heads/main" } } }
```

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with: { role-to-assume: arn:aws:iam::111122223333:role/gh-deploy, aws-region: eu-west-1 }
```

**Example 3 — GitHub Actions runs dbt or Terraform against Google Cloud.**

```bash
gcloud iam workload-identity-pools create github --location=global
gcloud iam workload-identity-pools providers create-oidc contoso --location=global --workload-identity-pool=github \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='contoso/data-platform' && assertion.ref=='refs/heads/main'"
gcloud iam service-accounts add-iam-policy-binding sa-cicd@contoso-data-prod.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/<NUM>/locations/global/workloadIdentityPools/github/attribute.repository/contoso/data-platform"
```

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/<NUM>/locations/global/workloadIdentityPools/github/providers/contoso
    service_account: sa-cicd@contoso-data-prod.iam.gserviceaccount.com
```

**Example 4 — A Google Cloud Run job writes to Azure Blob Storage (GCP → Azure).**

- *Why:* the job already has a Google SA. Google issues it **ID tokens**, which Entra can trust, so no Azure secret is needed in GCP.
- *Create:* on an Entra app registration (or user-assigned MI), add a federated credential with issuer `https://accounts.google.com`, subject = the SA's **unique numeric ID** (`gcloud iam service-accounts describe <sa> --format='value(uniqueId)'`) and audience `api://AzureADTokenExchange`. Grant the app **Storage Blob Data Contributor** on the container.
- *Use:* in the job, fetch a Google ID token with audience `api://AzureADTokenExchange` from the metadata server. Pass it to `azure.identity.ClientAssertionCredential(tenant_id, client_id, func=get_google_id_token)`, then use that credential with `BlobServiceClient`.

**Example 5 — AKS pods use a managed identity (Microsoft Entra Workload ID).**

- *Why:* pods should each have their own Azure identity, without secrets in Kubernetes Secrets.

```bash
az aks update -g rg-aks -n aks-prod --enable-oidc-issuer --enable-workload-identity
ISSUER=$(az aks show -g rg-aks -n aks-prod --query oidcIssuerProfile.issuerUrl -o tsv)
az identity create -g rg-aks -n id-orders-api
az identity federated-credential create -g rg-aks --identity-name id-orders-api -n k8s-orders \
  --issuer $ISSUER --subject system:serviceaccount:orders:orders-api --audiences api://AzureADTokenExchange
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-api
  namespace: orders
  annotations: { azure.workload.identity/client-id: "<client-id of id-orders-api>" }
---
# in the Deployment's pod template:
#   metadata.labels: { azure.workload.identity/use: "true" }
#   spec.serviceAccountName: orders-api
```

`DefaultAzureCredential` in the pod then picks up the projected token automatically.

### 8.3 Troubleshooting federation

| Error | Usual cause |
| --- | --- |
| `AADSTS70021: No matching federated identity record` | Subject doesn't match exactly (branch vs environment vs tag, case, `pull_request`) |
| AWS `Not authorized to perform sts:AssumeRoleWithWebIdentity` | `sub` or `aud` condition mismatch; workflow missing `id-token: write` |
| GCP `The given credential is rejected by the attribute condition` | Attribute condition too strict, or the wrong claim name |
| Token works in `main` but not in pull requests | That's by design. Don't widen the trust to PRs for production identities |

## Module 9 — Data-platform identities: Fabric, Databricks, Snowflake, BigQuery, GitHub

Data platforms reuse the cloud concepts above, but each has its own names and switches. This table is the translation key; the five examples show one end-to-end setup per platform.

### 9.1 Translation table

| Platform | Human identity | Workload identity | Best credential | Avoid | Where permissions live |
| --- | --- | --- | --- | --- | --- |
| **Microsoft Fabric / Power BI** | Entra user / guest | Entra **service principal**, **managed identity**, Fabric **workspace identity** | MI (in Azure) or federated credential / certificate | Personal accounts in pipelines and dataset credentials | Tenant settings, workspace roles, item permissions, SQL GRANT, RLS |
| **Databricks** | Users synced from IdP via SCIM | Databricks **service principal** (account level) or Entra SP on Azure | **OAuth M2M** (secret) or **OAuth token federation** | **PATs** of humans | Unity Catalog grants, workspace entitlements, ACLs on warehouses, jobs and apps |
| **Snowflake** | `TYPE = PERSON` users via SSO / SCIM | `TYPE = SERVICE` users | **Key-pair JWT**, **workload identity federation**, External OAuth | Passwords for service users | Roles (RBAC) + policies |
| **BigQuery** | Google accounts / Workforce IF | Google **service account** | Attached SA, impersonation, WIF | SA JSON keys | IAM on project, dataset or table + row and column policies |
| **GitHub** | GitHub user + SSO (Enterprise) | **GitHub App** or the Actions `GITHUB_TOKEN` | GitHub App (short-lived installation tokens) | Classic PATs with broad `repo` scope | Repo, org and team permissions; fine-grained token permissions |

### 9.2 Five real examples

**Example 1 — Fabric: a notebook reads a firewalled ADLS account (workspace identity).**

- *Why:* the storage account blocks public access. Fabric must reach it with an identity, not a SAS token.
- *Create:* workspace settings → **Workspace identity → + Workspace identity** (Fabric creates a managed-identity-style SP named after the workspace). On the storage account, grant it **Storage Blob Data Reader**, and add a **resource instance rule** for trusted workspace access.
- *Use:* create an ADLS shortcut or pipeline connection with authentication type **Workspace identity**. No secret exists. For Fabric REST automation from outside, use an SP or MI added to `sg-fabric-api-clients` (Module 4, example 2).

**Example 2 — Databricks: a job runs as a service principal with OAuth M2M.**

- *Why:* jobs owned by a person break when they leave and run with their (too-broad) rights.

```bash
# Account console -> User management -> Service principals -> Add: sp-etl-prod -> Secrets -> Generate OAuth secret
databricks jobs update --json '{"job_id": 123, "new_settings": {"run_as": {"service_principal_name": "<sp-application-id>"}}}'
```

```sql
GRANT USE CATALOG ON CATALOG contoso_prod TO `<sp-application-id>`;
GRANT USE SCHEMA, SELECT, MODIFY ON SCHEMA contoso_prod.gold TO `<sp-application-id>`;
```

External tools authenticate with `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET` (the SDK and CLI do OAuth M2M automatically). Disable PAT creation for users in workspace settings.

**Example 3 — Snowflake: Fivetran or Airflow loads data as a key-pair service user.**

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes-256-cbc -inform PEM -out loader_key.p8   # encrypted private key
openssl rsa -in loader_key.p8 -pubout -out loader_key.pub
```

```sql
CREATE ROLE R_LOADER;
GRANT USAGE ON WAREHOUSE WH_LOAD TO ROLE R_LOADER;
GRANT USAGE, CREATE TABLE ON SCHEMA RAW.SALES TO ROLE R_LOADER;
CREATE USER SVC_LOADER TYPE = SERVICE DEFAULT_ROLE = R_LOADER DEFAULT_WAREHOUSE = WH_LOAD
  RSA_PUBLIC_KEY = '<body of loader_key.pub>';
GRANT ROLE R_LOADER TO USER SVC_LOADER;
```

Upload the private key (and its passphrase) to the tool, or store them in your vault and reference them. Rotate using the `RSA_PUBLIC_KEY_2` slot.

**Example 4 — BigQuery: Looker Studio dashboards use viewer credentials, not an owner's.**

- *Why:* a data source set to *Owner's credentials* lets every report viewer see whatever the owner sees, which bypasses row-level security.
- *Setup:* in the Looker Studio data source, set **Data credentials → Viewer**. Viewers then query BigQuery as themselves, and IAM, row access policies and policy tags apply per person. For scheduled jobs, use a dedicated SA with *Service account credentials* (Looker Studio's service agent granted `serviceAccountTokenCreator` on that SA). Never use a personal account.

**Example 5 — GitHub: automation uses a GitHub App instead of a personal access token.**

- *Why:* a PAT acts as *you*, across all your repos, often for months. A GitHub App is installed on chosen repos with chosen permissions, and issues **1-hour installation tokens**.
- *Create:* Org settings → Developer settings → **GitHub Apps → New**. Set permissions (for example *Contents: read*, *Pull requests: write*), generate a **private key** (store it in your vault), then **Install** the app on the selected repos only.
- *Use in Actions:*

```yaml
- uses: actions/create-github-app-token@v1
  id: app
  with: { app-id: ${{ vars.APP_ID }}, private-key: ${{ secrets.APP_PRIVATE_KEY }} }
- run: gh pr create --title "Nightly docs" --body "auto"
  env: { GH_TOKEN: ${{ steps.app.outputs.token }} }
```

Within a single repo's workflow, prefer the built-in `GITHUB_TOKEN` with an explicit `permissions:` block. If you really need a PAT, use a **fine-grained PAT** limited to specific repos, with an expiry.

## Module 10 — OAuth 2.0 and OpenID Connect

**OAuth 2.0** is the standard way for an app to get an **access token** to call an API, either for a user (delegated) or as itself (app-only), without ever seeing the user's password. **OpenID Connect (OIDC)** is a thin layer on top that adds an **ID token**, so the app also learns *who* signed in. Entra ID, Google, Okta, Auth0, AWS Cognito, GitHub, Databricks and Snowflake all speak these protocols. The names differ, but the flows are identical.

### 10.1 The four roles

| Role | Who | Example |
| --- | --- | --- |
| **Resource owner** | The user who owns the data | Sarah |
| **Client** | The app that wants access (has a **Client ID**) | Your web app, a script, Power BI |
| **Authorization server** | Issues tokens after checking identity and consent | Entra ID (`login.microsoftonline.com`), Google (`accounts.google.com`) |
| **Resource server** | The API that accepts tokens | Microsoft Graph, Fabric API, your FastAPI |

Useful endpoints come from the **discovery document**: `https://login.microsoftonline.com/<tenant>/v2.0/.well-known/openid-configuration` (authorize, token and JWKS signing-keys URLs).

### 10.2 Choosing a flow

| Flow | User present? | Client type | Use for | Status |
| --- | --- | --- | --- | --- |
| **Authorization code + PKCE** | Yes | Web, SPA, mobile, desktop | Sign-in and calling APIs as the user | **Recommended** |
| **Client credentials** | No | Confidential (has secret, cert or federation) | Daemons, jobs, service-to-service | **Recommended** |
| **Device code** | Yes, on another device | CLI, TV, headless server | `az login --use-device-code`, IoT | OK; phishing-prone, so restrict with CA |
| **On-behalf-of (OBO)** / token exchange | Yes (upstream) | API calling another API | Middle-tier API keeps the user's identity downstream | Recommended where needed |
| **Refresh token** | No (after first sign-in) | Any client that got one | Getting new access tokens silently | Used by all libraries |
| Implicit | Yes | SPA | — | **Deprecated**, use code + PKCE |
| Resource owner password (ROPC) | Yes | Any | — | **Avoid**: breaks MFA, sends passwords to the app |

### 10.3 Authorization code + PKCE, step by step

&#91;embedded content: Authorization code + PKCE · 4 parties, 8 steps\]

The password and MFA go only to Entra (step 3). The app swaps a one-time code plus its PKCE verifier for tokens (steps 5–6), and the API sees only the access token (step 7), which it validates before returning data.

**PKCE** (Proof Key for Code Exchange): the app invents a random `code_verifier` and sends its hash (`code_challenge`) with the login request. When swapping the code for tokens, it must present the original verifier, so a stolen authorization code is useless to anyone else.

### 10.4 Five real examples

**Example 1 — Internal Streamlit or FastAPI web app signs users in (auth code + PKCE).**

```python
# pip install msal
import msal
app = msal.ConfidentialClientApplication("<client-id>", authority="https://login.microsoftonline.com/<tenant>",
                                         client_credential="<secret-or-cert>")
flow = app.initiate_auth_code_flow(scopes=["User.Read"], redirect_uri="https://app.contoso.com/callback")
# redirect browser to flow["auth_uri"]; on callback:
result = app.acquire_token_by_auth_code_flow(flow, request_args_from_callback)
id_claims = result["id_token_claims"]          # who: name, preferred_username, oid, tid
access_token = result["access_token"]           # for Graph User.Read
```

MSAL generates the state, nonce and PKCE values. Store the token cache server-side, never in browser local storage.

**Example 2 — Nightly job calls an API as itself (client credentials).**

```bash
curl -s -X POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token \
  -d grant_type=client_credentials -d client_id=<client-id> \
  -d client_secret="$SECRET" -d scope=api://api-sales/.default
# -> { "token_type": "Bearer", "expires_in": 3599, "access_token": "eyJ..." }
```

There's no user and no refresh token: just request a new token when the old one expires. The same flow exists on Databricks (`/oidc/v1/token`), Okta, Auth0 and Google (via SA JWT).

**Example 3 — Signing in on a headless server (device code).**

```bash
az login --use-device-code
# "To sign in, use a web browser to open https://microsoft.com/devicelogin and enter the code ABCD-EFGH"
```

You complete MFA on your phone or laptop, and the server receives tokens. Attackers abuse this ("enter this code to see the invoice"), so block device code flow with Conditional Access for everyone who doesn't need it.

**Example 4 — Your API calls Microsoft Graph or Fabric as the signed-in user (on-behalf-of).**

- *Why:* the API must read *Sarah's* OneDrive or *Sarah's* Fabric items, with Sarah's permissions, not with a powerful app-only permission.

```python
obo = app.acquire_token_on_behalf_of(user_assertion=incoming_bearer_token,
                                     scopes=["https://api.fabric.microsoft.com/.default"])
```

The incoming token must have been issued for *your* API (audience `api://api-sales`), and your API's app registration needs delegated permissions to the downstream API.

**Example 5 — A Python script updates a Google Sheet for one user (installed-app flow + refresh token).**

- *Why:* the sheet belongs to a person, so the script must act as them with consent, once, then keep working unattended.
- *Create:* Google Cloud console → **OAuth consent screen** (Internal if Workspace), then **Credentials → OAuth client ID → Desktop app**. Download `client_secret.json`. This holds a client secret, but for desktop apps Google treats it as non-confidential, so the security rests on user consent and the refresh token.

```python
from google_auth_oauthlib.flow import InstalledAppFlow
flow = InstalledAppFlow.from_client_secrets_file("client_secret.json",
        scopes=["https://www.googleapis.com/auth/spreadsheets"])
creds = flow.run_local_server(port=0)        # browser consent once
open("token.json", "w").write(creds.to_json())  # contains the REFRESH TOKEN -> treat as a secret, store in a vault
```

Refresh tokens for apps in *Testing* publishing status expire after 7 days. Publish the consent screen (or use Internal) for long-running jobs. For server-side automation on Workspace data, a service account with domain-wide delegation, or simply sharing the sheet with the SA's email, avoids user tokens entirely.

### 10.5 Common OAuth mistakes

- Asking for `.default` or `*.ReadWrite.All` when one narrow scope would do.
- Putting a client secret in a SPA or mobile app. Public clients can't keep secrets, so use PKCE without a secret.
- Not validating `state` (CSRF) and `nonce` (replay). Libraries do this; hand-rolled code often doesn't.
- Logging access or refresh tokens. They're bearer credentials: whoever holds one can use it.

## Module 11 — Tokens: JWTs, access, ID and refresh tokens, API keys, PATs, SAS

A **token** is a credential with a built-in expiry. Most are **bearer** tokens: whoever holds one can use it, like cash. Handle every token as a secret, keep lifetimes short, and scope it to one audience and as few permissions as possible.

### 11.1 Token types at a glance

| Token | Issued by | Proves | Typical lifetime | Sent as |
| --- | --- | --- | --- | --- |
| **Access token** | Authorization server (Entra, Google, Okta, Databricks…) | Caller may call **this API** with **these scopes/roles** | 60–90 min (Entra), 1 h (Google, AWS STS default) | `Authorization: Bearer <token>` |
| **ID token** (OIDC) | Authorization server | **Who** signed in, for the *client app* | \~1 h | Never sent to APIs; read by the app |
| **Refresh token** | Authorization server | Right to get new access tokens without a new sign-in | Days to 90 days, sliding | Only to the token endpoint |
| **API key** | The API provider | "This caller is a known customer" (usually no user) | Until revoked | Header such as `x-api-key` |
| **Personal access token (PAT)** | GitHub, Databricks, Snowflake, Azure DevOps | Acts as **a user** | Days to a year | `Authorization: Bearer` / `token` |
| **SAS token** (Azure Storage) | Signed with a storage key or user delegation key | Specific permissions on a container or blob until an expiry | Minutes to days | URL query string `?sv=...&sig=...` |
| **Presigned URL** (S3, GCS) | Signed with IAM credentials | One object, one operation, until expiry | Max 7 days | URL query string |

### 11.2 Anatomy of a JWT

A JSON Web Token is three Base64URL parts: `header.payload.signature`.

```json
// header
{ "typ": "JWT", "alg": "RS256", "kid": "Xr9f..." }          // kid = which public key signed it
// payload (claims) of an Entra access token for your API
{
  "aud": "api://api-sales",                 // audience: the API this token is FOR
  "iss": "https://login.microsoftonline.com/<tenant>/v2.0",   // who issued it
  "iat": 1791800000, "nbf": 1791800000, "exp": 1791803599,  // issued / not before / expires (Unix time)
  "tid": "<tenant-id>",
  "oid": "<object-id of user or SP>",       // stable id: use this for RLS mappings
  "sub": "<pairwise subject>",
  "azp": "<client-id of the calling app>",  // appid in v1 tokens
  "scp": "Sales.Read",                      // delegated scopes (user tokens)
  "roles": ["Sales.Read.All"],              // app roles (app-only tokens) or user app roles
  "ver": "2.0"
}
// signature = RS256( base64url(header) + "." + base64url(payload), issuer's private key )
```

Anyone can **read** a JWT (it's encoded, not encrypted). Only the issuer can **sign** one, so validation means checking the signature with the issuer's **public** keys (JWKS) plus the claims.

### 11.3 Five real examples

**Example 1 — Debugging a 401: decode the token you're sending.**

```python
import jwt   # pip install pyjwt
claims = jwt.decode(token, options={"verify_signature": False})   # inspection only!
print(claims["aud"], claims["iss"], claims.get("scp"), claims.get("roles"), claims["exp"])
```

Check these: `aud` matches the API (a Graph token sent to Fabric fails); `iss` / `tid` is the right tenant; `exp` is in the future; `scp` or `roles` holds what the API needs. You can paste tokens into **jwt.ms** (Microsoft) to decode them in the browser, but only test tokens, never production ones.

**Example 2 — Validate Entra tokens in your FastAPI.**

```python
import jwt
from jwt import PyJWKClient
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

TENANT = "<tenant-id>"
AUDIENCE = "api://api-sales"
ISSUER = f"https://login.microsoftonline.com/{TENANT}/v2.0"
jwks = PyJWKClient(f"https://login.microsoftonline.com/{TENANT}/discovery/v2.0/keys")   # cached

def caller(creds: HTTPAuthorizationCredentials = Security(HTTPBearer())) -> dict:
    try:
        key = jwks.get_signing_key_from_jwt(creds.credentials).key
        claims = jwt.decode(creds.credentials, key, algorithms=["RS256"],
                            audience=AUDIENCE, issuer=ISSUER, leeway=60)
    except jwt.PyJWTError as e:
        raise HTTPException(401, f"invalid token: {e}")
    if "Sales.Read.All" not in claims.get("roles", []) and "Sales.Read" not in claims.get("scp", "").split():
        raise HTTPException(403, "missing Sales.Read permission")
    return claims        # use claims["oid"] to look up the client / RLS mapping
```

Never accept `alg: none`, never skip `aud`, and always pin the expected algorithm. (On Azure App Service and Container Apps, **Easy Auth** can do this validation in front of your code.)

**Example 3 — A partner uploads files for 24 hours (user delegation SAS).**

- *Why:* the partner has no Azure identity in your tenant. A SAS grants exactly *write* to one container, until tomorrow. A **user delegation SAS** is signed with an Entra-issued key, not the storage account key, so it can be revoked and audited per identity.

```bash
az storage container generate-sas --account-name contosolake --name partner-uploads \
  --permissions cw --expiry 2026-10-14T18:00Z --https-only \
  --auth-mode login --as-user -o tsv
# -> se=...&sp=cw&sv=...&sr=c&skoid=...&sig=...   send https://contosolake.blob.core.windows.net/partner-uploads?<sas>
```

Avoid **account SAS**, which is signed with the account key, covers everything, and can only be revoked by rotating the key.

**Example 4 — Designing API keys for your own API.**

- Generate 32+ random bytes with a recognisable prefix (`cds_live_...`) so secret scanners can detect leaks.
- Store only a **hash** (SHA-256) of the key, plus its owner, scopes, creation date and expiry. Show the full key once.
- Support **two active keys per client** for zero-downtime rotation, and log the last-used time.
- Rate-limit per key, and never accept keys in URLs (they end up in logs). Use a header.
- For anything beyond simple metering, put a gateway (APIM, Apigee, AWS API Gateway) in front. It manages keys, quotas and revocation for you.

**Example 5 — When a PAT is acceptable, and how to make it safer.**

- *Acceptable:* a quick personal script on your laptop, or a tool that supports nothing else.
- *Safer PAT recipe:*
  - GitHub **fine-grained** token → only selected repos, only *Contents: read*, 30-day expiry.
  - Databricks PAT → created for a **service principal** (not a person) through the token management API, with `lifetime_seconds` set; admins restrict who may create tokens.
  - Snowflake PAT → `ROLE_RESTRICTION` + `DAYS_TO_EXPIRY` + a network policy on the user.
- Store it in a vault or password manager, never in `.env` files committed to Git. Turn on **GitHub secret scanning + push protection** to block accidental commits.

### 11.4 Token lifetime and revocation facts

- Revoking a user or SP's access **doesn't kill access tokens already issued**. They stay valid until `exp` (up to about 1 h). Entra **Continuous Access Evaluation** shortens this for supporting services (Exchange, SharePoint, Graph) when a user is disabled or the location changes.
- To cut access fast, disable the account (stops refresh), revoke sessions (`az rest -m POST -u https://graph.microsoft.com/v1.0/users/<id>/revokeSignInSessions`) and rotate any secrets it held.
- Keep tokens in memory, use the library's token cache, and never write them to logs, notebooks or screenshots.

## Module 12 — Secret vaults on every platform

A **vault** stores secrets (passwords, API keys, connection strings), **keys** (for encryption and signing) and **certificates**. Access is controlled by identity, every read is logged, and versioning plus rotation are supported. The pattern is always the same: *the workload's identity (Modules 4–8) reads the secret at runtime*. Secrets never sit in code, config files, notebooks or CI variables in plain text.

### 12.1 Secrets vs keys vs certificates

| Object | What it is | You get back | Example |
| --- | --- | --- | --- |
| **Secret** | Any sensitive string | The value itself | Partner API key, SQL password, refresh token |
| **Key** | Cryptographic key that (ideally) never leaves the vault or HSM | Encrypt, decrypt, sign or verify **operations**, not the key | CMEK for storage, signing JWTs, Tri-Secret Secure |
| **Certificate** | X.509 cert + private key, with renewal policy | Cert (public) and, if exportable, the private key | SP certificate (Module 3), TLS for an API |

### 12.2 The vaults compared

| Platform | Secrets | Keys | Access model | Rotation | Notable features |
| --- | --- | --- | --- | --- | --- |
| **Azure Key Vault** (+ Managed HSM) | Yes | Yes (software / HSM) | **Azure RBAC** (recommended) or legacy access policies | Key rotation policies; Event Grid *near expiry* events for secrets | Soft delete + **purge protection**, private endpoints, references from App Service, Functions, ADF, Fabric connections, Databricks scopes |
| **AWS Secrets Manager** | Yes | (uses KMS) | IAM + resource policies | **Built-in rotation** with Lambda (native for RDS, Redshift, DocumentDB) | Cross-region replication, ECS/Lambda/EKS integrations |
| **AWS SSM Parameter Store** | SecureString parameters | (uses KMS) | IAM | Manual | Cheap for config + simple secrets |
| **AWS KMS** | — | Yes | Key policies + IAM + grants | Automatic yearly for AWS-managed rotation | Envelope encryption for every AWS service |
| **GCP Secret Manager** | Yes (versioned) | (uses Cloud KMS for CMEK) | IAM (`secretmanager.secretAccessor`) | Rotation **schedules** emitting Pub/Sub events to your rotator | Regional replication control, mount as file or env var in Cloud Run/GKE |
| **GCP Cloud KMS** (+ HSM, EKM) | — | Yes | IAM | Automatic rotation period | External Key Manager keeps keys outside Google |
| **HashiCorp Vault** (self-managed / HCP) | Yes, plus **dynamic secrets** | Transit engine | Policies + auth methods (OIDC, Kubernetes, AWS, Azure, JWT) | Leases expire automatically | Generates short-lived DB/cloud creds on demand; multi-cloud |
| **Databricks secret scopes** | Yes | — | Scope ACLs (READ/WRITE/MANAGE) | Via backing store | **Azure Key Vault-backed** scopes; values redacted in notebook output |
| **GitHub Actions secrets** | Yes (repo / environment / org) | — | Repo permissions; environment protection rules | Manual | Masked in logs; prefer OIDC (Module 8) over long-lived secrets |
| **Snowflake** `SECRET` objects | Yes | (Tri-Secret Secure for keys) | Roles (`USAGE` on secret) | Manual | Used by external access integrations in UDFs and procedures |

### 12.3 Five real examples

**Example 1 — Azure Key Vault done right, consumed by an App Service and a Fabric connection.**

```bash
az keyvault create -n kv-contoso-prod -g rg-sec -l westeurope \
  --enable-rbac-authorization true --enable-purge-protection true --retention-days 90 \
  --public-network-access Disabled            # then add a private endpoint
az role assignment create --assignee <web-app-mi-principal-id> --role "Key Vault Secrets User" \
  --scope $(az keyvault show -n kv-contoso-prod --query id -o tsv)/secrets/partner-api-key
az keyvault secret set --vault-name kv-contoso-prod -n partner-api-key --value "<value>" --expires 2027-04-01T00:00:00Z
```

- App Service setting: `PARTNER_API_KEY=@Microsoft.KeyVault(VaultName=kv-contoso-prod;SecretName=partner-api-key)`. The app reads it as a normal env var, and the platform fetches it with the app's managed identity.
- Fabric: where a connection type supports **Azure Key Vault references**, point the connection's password at the vault secret instead of typing it in.
- Grant at the **individual secret** scope when an app needs one secret, not the whole vault.

**Example 2 — AWS Secrets Manager rotates an RDS PostgreSQL password automatically.**

```bash
aws secretsmanager create-secret --name prod/sales-db/app \
  --secret-string '{"username":"app","password":"<initial>","engine":"postgres","host":"sales.xxxx.rds.amazonaws.com","port":5432,"dbname":"sales"}'
aws secretsmanager rotate-secret --secret-id prod/sales-db/app \
  --rotation-lambda-arn <arn of the AWS-provided PostgreSQL single-user rotation function> \
  --rotation-rules AutomaticallyAfterDays=30
```

Apps fetch with `get_secret_value` each time they connect (or cache it briefly), so rotation needs no redeploy. For RDS you can also let RDS manage the master password in Secrets Manager (`--manage-master-user-password`), or use **IAM database authentication** and have no password at all.

**Example 3 — GCP Secret Manager mounted into Cloud Run.**

```bash
printf '%s' "<value>" | gcloud secrets create partner-api-key --data-file=- --replication-policy=user-managed --locations=europe-west1
gcloud secrets add-iam-policy-binding partner-api-key \
  --member=serviceAccount:sa-sales-api@contoso-api-prod.iam.gserviceaccount.com --role=roles/secretmanager.secretAccessor
gcloud run services update sales-api --set-secrets=PARTNER_API_KEY=partner-api-key:latest      # as env var
# or as a file: --set-secrets=/secrets/partner/key=partner-api-key:latest
```

Pin a version number (`:3`) for reproducible deploys, or use `:latest` with a file mount so new versions are picked up without redeploying.

**Example 4 — Databricks notebook uses a Key Vault-backed secret scope.**

```bash
databricks secrets create-scope kv-prod --scope-backend-type AZURE_KEYVAULT \
  --resource-id <key-vault-resource-id> --dns-name https://kv-contoso-prod.vault.azure.net/
databricks secrets put-acl kv-prod grp-data-engineers READ
```

```python
api_key = dbutils.secrets.get(scope="kv-prod", key="partner-api-key")
print(api_key)          # prints [REDACTED]
```

Redaction only stops *accidental* display. Anyone with READ on the scope can still exfiltrate the value with code, so scope ACLs are the real control. For storage access in Unity Catalog, use storage credentials (Module 5, example 5) instead of secrets.

**Example 5 — HashiCorp Vault issues dynamic database credentials.**

- *Why:* instead of one long-lived DB password shared by all instances, each app instance gets its own username and password that expire in 1 hour and are created on demand.

```bash
vault secrets enable database
vault write database/config/sales plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@sales-db:5432/sales" \
  allowed_roles="app-readonly" username="vault_admin" password="<admin>"
vault write database/roles/app-readonly db_name=sales default_ttl=1h max_ttl=4h \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
# app (authenticated to Vault via its Kubernetes SA or cloud identity):
vault read database/creds/app-readonly      # -> unique username/password, lease 1h
```

The app authenticates to Vault with its platform identity (Kubernetes auth, Azure MI, AWS IAM), so the chain has no static secret anywhere.

### 12.4 Vault hygiene checklist

- [ ] One vault per application per environment (blast radius), never a shared "company vault".
- [ ] RBAC with least privilege: apps get *read secret* on their own secrets only, and humans don't get read on prod secrets by default.
- [ ] Soft delete + purge protection on; private network access; diagnostic logs to your SIEM.
- [ ] Every secret has an **expiry date and an owner**, with an alert 30 days before expiry.
- [ ] Rotation automated where the platform supports it; quarterly manual rotation otherwise.
- [ ] No secrets in Git, notebooks, Slack, email, tickets or screenshots. Secret scanning on all repos.

## Module 13 — Authorization in depth: RBAC, ABAC, RLS, OLS and least-privilege roles

Authentication (Modules 2–12) gets an identity through the door. **Authorization** decides what it may touch, at four levels of detail: the **resource** (RBAC on a workspace, database, bucket), the **object** (a table, column or measure, via OLS and CLS), the **row** (RLS) and the **value** (masking). Least privilege means applying the narrowest level that still lets the job get done.

### 13.1 The models

| Model | Decides by | Example | Strength | Weakness |
| --- | --- | --- | --- | --- |
| **RBAC** (role-based) | Which **role** the principal holds on which **scope** | Group `grp-analysts` = *Reader* on resource group `rg-sales` | Simple, auditable | Role explosion when rules depend on data |
| **ABAC** (attribute-based) | **Attributes** of principal, resource and context | "Read blobs only where tag `project` = the user's `project` attribute" | Scales with fewer roles | Harder to reason about; attributes must be trustworthy |
| **ReBAC** (relationship-based) | Relationships in a graph | "Owner of the document's folder may edit" | Natural for sharing apps | Needs a dedicated engine |
| **RLS** (row-level security) | A filter predicate per identity | Client 1 sees only `client_id = 1` | Same table, different rows | Admin roles often bypass it |
| **CLS / OLS** (column / object-level) | Hide or deny columns, tables or measures | Hide `salary`, hide the `HR` table | Removes sensitive fields entirely | Queries that reference hidden objects fail |
| **Dynamic data masking** | Transforms values per identity | `a***@contoso.com` | Users still see shape and format | Cosmetic in some engines; not a boundary |

### 13.2 Least privilege, in five rules

1. **Grant to groups or roles, never to individuals.** Membership changes; grants stay stable.
2. **Narrowest scope:** resource > resource group > subscription; table > schema > database; dataset > project.
3. **Smallest built-in role that works,** before inventing a custom one (see 13.3), and never *Owner*, *Admin* or *Editor* for workloads.
4. **Time-bound elevation** (PIM, PAM, IAM Identity Center sessions) for anything admin.
5. **Measure and trim:** use unused-permission reports (Entra access reviews, AWS IAM Access Analyzer, GCP IAM Recommender, Snowflake `GRANTS_TO_ROLES` + `ACCESS_HISTORY`, Databricks `system.access.audit`).

### 13.3 Least-privilege role cheat sheet

| Task | Azure | AWS | GCP | Fabric / Power BI | Databricks | Snowflake |
| --- | --- | --- | --- | --- | --- | --- |
| Read files in a lake | Storage Blob Data **Reader** on the container | `s3:GetObject` on `bucket/prefix/*` | `roles/storage.objectViewer` on the bucket | Lakehouse **Read** + ReadAll, or a OneLake security role | `READ FILES` on an external location / `READ VOLUME` | `USAGE` on a stage |
| Query tables | SQL `db_datareader` or `GRANT SELECT ON SCHEMA` | Athena/Redshift grants + `s3:GetObject` | `roles/bigquery.dataViewer` on the dataset + `jobUser` | Warehouse **Read** + `GRANT SELECT` | `USE CATALOG` + `USE SCHEMA` + `SELECT` | `USAGE` (db, schema, wh) + `SELECT` |
| Run pipelines, not edit them | *Data Factory Contributor* is too much; custom role with `pipelines/createRun/action` | `states:StartExecution` on one state machine | `roles/composer.user` / `workflows.invoker` | Workspace **Contributor** (no narrower role yet) or item-level Execute | `CAN MANAGE RUN` on the job | `OPERATE` on the task |
| Read one secret | **Key Vault Secrets User** on that secret | `secretsmanager:GetSecretValue` on one ARN | `roles/secretmanager.secretAccessor` on one secret | (via Key Vault reference) | `READ` on the scope | `USAGE` on the secret |
| View reports only | n/a | n/a | Looker *Viewer* | Workspace **Viewer** or app audience | `CAN VIEW` dashboard | n/a |
| Manage access, not data | *User Access Administrator* with a **condition** limiting assignable roles | IAM with a permission boundary | `roles/iam.securityAdmin` (scoped) | Workspace **Member** (can share) | `MANAGE` privilege | `SECURITYADMIN` or `MANAGE GRANTS` |

### 13.4 RLS and OLS on each platform

| Platform | Row-level | Column / object-level | Masking | Who bypasses |
| --- | --- | --- | --- | --- |
| **Power BI / Fabric semantic model** | Roles with DAX filters; dynamic with `USERPRINCIPALNAME()` | **OLS** (tables, columns) via Tabular Editor or TMDL | — | Workspace Admin/Member/Contributor; dataset Write |
| **Fabric Warehouse / SQL Server / Azure SQL** | `CREATE SECURITY POLICY` + predicate function | `GRANT SELECT (cols)`, `DENY` | `MASKED WITH (FUNCTION=...)` | `db_owner`, `UNMASK` |
| **Fabric OneLake security** | Role row filter (T-SQL predicate) | Role column security | — | Workspace Admin/Member/Contributor |
| **Databricks Unity Catalog** | `SET ROW FILTER` / ABAC policies | Column masks; don't grant | Column mask functions | Owners/admins per policy design |
| **Snowflake** | Row access policies | Masking + projection policies; secure views | Masking policies, tag-based | Policy logic (e.g. an admin role) |
| **BigQuery** | `CREATE ROW ACCESS POLICY` | Policy tags (fine-grained reader) | Data policies (`EMAIL_MASK`, `SHA256`…) | Grantees of `FILTER USING (TRUE)` |

### 13.5 Five real examples

**Example 1 — Power BI: managers see their own region, HR columns hidden from everyone but HR.**

- *Why:* one report for 40 regional managers instead of 40 copies; salary columns invisible, not just blank.
- *RLS (dynamic):* table `UserRegion[upn, region]` related to `Region`. Role **Manager**, filter on `UserRegion`:

```dax
'UserRegion'[upn] = USERPRINCIPALNAME()
```

- *OLS:* in Tabular Editor or TMDL, role **NonHR** gets `columnPermission Employee.salary = none` and `tablePermission Compensation = none`.

```
role NonHR
    modelPermission: read
    tablePermission Employee
        columnPermission salary = none
    tablePermission Compensation = none
```

- *Assign:* Entra group `grp-regional-managers` → roles Manager + NonHR. Managers must be workspace **Viewers** (or app users). If they're Contributors, RLS doesn't apply.
- *Test:* **Model → View as → Other user** = a manager's UPN.

**Example 2 — Fabric Warehouse / Azure SQL: RLS + column grants + masking together.**

```sql
CREATE SCHEMA sec;
GO
CREATE FUNCTION sec.fn_region(@region VARCHAR(20)) RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS ok FROM dbo.user_region ur
       WHERE ur.region = @region AND ur.upn = USER_NAME()
       UNION ALL SELECT 1 WHERE IS_MEMBER('grp-finance-all') = 1;
GO
CREATE SECURITY POLICY sec.region_policy ADD FILTER PREDICATE sec.fn_region(region) ON dbo.sales WITH (STATE = ON);

GRANT SELECT ON dbo.sales (sale_id, region, sale_date, amount) TO [grp-regional-managers];   -- CLS
ALTER TABLE dbo.customers ALTER COLUMN phone ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');
```

**Example 3 — Azure: a custom "pipeline operator" role plus an ABAC condition.**

- *Why:* the support team must re-run failed pipelines but never edit or delete them. *Data Factory Contributor* would let them change everything.

```json
{
  "Name": "ADF Pipeline Operator",
  "IsCustom": true,
  "Description": "Run and monitor pipelines, no edits",
  "Actions": [
    "Microsoft.DataFactory/factories/read",
    "Microsoft.DataFactory/factories/pipelines/read",
    "Microsoft.DataFactory/factories/pipelines/createrun/action",
    "Microsoft.DataFactory/factories/pipelineruns/read",
    "Microsoft.DataFactory/factories/pipelineruns/cancel/action"
  ],
  "AssignableScopes": ["/subscriptions/<sub>/resourceGroups/rg-data-prod"]
}
```

```bash
az role definition create --role-definition adf-operator.json
```

**ABAC on storage:** grant *Storage Blob Data Reader* with a **condition**, so the analytics team reads only blobs tagged `classification = public`:

```bash
az role assignment create --assignee <group-object-id> --role "Storage Blob Data Reader" --scope <storage-account-id> \
  --condition "((!(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'})) OR (@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:classification<\$key_case_sensitive\$>] StringEquals 'public'))" \
  --condition-version "2.0"
```

**Example 4 — AWS: generate a least-privilege policy from real usage, then cap it with a boundary.**

- *Why:* the ETL role started with `AmazonS3FullAccess` "temporarily". Six months later, it still has it.
- *Steps:*
  1. **IAM Access Analyzer → Generate policy** from 90 days of CloudTrail for `etl-role`. It proposes only the actions and resources actually used.
  2. Review it, replace the managed policy, and run for a week.
  3. Add a **permission boundary** so even if someone later attaches a broad policy, the role can never exceed S3 and Glue in one account:

```json
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Action": ["s3:*", "glue:*"], "Resource": "*" },
  { "Effect": "Deny", "Action": ["s3:DeleteBucket", "s3:PutBucketPolicy"], "Resource": "*" } ] }
```

**Example 5 — The same rule on three warehouses: "a manager sees only their region".**

```sql
-- Databricks (Unity Catalog)
CREATE FUNCTION sec.rf_region(r STRING) RETURN is_account_group_member('finance-all')
  OR EXISTS (SELECT 1 FROM sec.user_region u WHERE u.upn = current_user() AND u.region = r);
ALTER TABLE gold.sales SET ROW FILTER sec.rf_region ON (region);

-- Snowflake
CREATE ROW ACCESS POLICY sec.rap_region AS (r STRING) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('R_FINANCE_ALL')
  OR EXISTS (SELECT 1 FROM sec.user_region u WHERE u.upn = CURRENT_USER() AND u.region = r);
ALTER TABLE gold.sales ADD ROW ACCESS POLICY sec.rap_region ON (region);

-- BigQuery
CREATE ROW ACCESS POLICY rap_region ON `proj.gold.sales`
GRANT TO ('group:regional-managers@contoso.com')
FILTER USING (region IN (SELECT region FROM `proj.sec.user_region` WHERE upn = SESSION_USER()));
```

The pattern is identical everywhere: a **mapping table** (who → which rows), a **function or policy** that compares it with the current identity, and an **escape hatch** for a privileged group. Keep the mapping table in a schema that ordinary users can't modify.

### 13.6 Authorization review checklist

- [ ] Every grant is to a group or role; individual grants are listed and justified.
- [ ] No workload holds Owner, Admin, Editor, `ACCOUNTADMIN`, `*:*` or `ALL PRIVILEGES`.
- [ ] Report consumers are Viewers (so RLS applies), not Contributors.
- [ ] Every table with personal or client data has RLS or a secure view, and PII columns have OLS, CLS or masking.
- [ ] RLS is tested automatically: one test user per role asserts the rows they can see.
- [ ] Unused permissions are reviewed quarterly, using the platform's recommender or analyzer.

## Module 14 — Cross-platform comparison and decision guide

The same seven ideas exist on every platform under different names. Use this module as your translation dictionary and your "which one do I pick?" flowchart.

### 14.1 Translation dictionary

| Concept | Microsoft Entra / Azure | AWS | Google Cloud | Databricks | Snowflake | GitHub |
| --- | --- | --- | --- | --- | --- | --- |
| Directory / tenant | Entra **tenant** | AWS **Organization** / account | Cloud Identity / Workspace **organization** | **Account** | **Organization** / account | **Enterprise / organization** |
| Human SSO | Entra ID (OIDC/SAML) | **IAM Identity Center** | Google accounts / **Workforce Identity Federation** | SSO + SCIM from IdP | SAML SSO + SCIM | SAML SSO (Enterprise) |
| Workload identity, platform-managed | **Managed identity** | **IAM role** (instance profile, task role, Lambda role) | **Attached service account** | Access connector (Azure MI) | (none; uses cloud roles for storage integrations) | `GITHUB_TOKEN` |
| Workload identity, self-managed | **Service principal** (app registration) | IAM role assumed with keys / IAM user | **Service account** | **Service principal** | `TYPE = SERVICE` user | **GitHub App** |
| Long-lived secret (avoid) | Client secret | **Access key** (`AKIA…`) | **SA JSON key** | OAuth secret / **PAT** | Password / PAT | Classic **PAT** |
| Stronger credential | Certificate | Roles Anywhere (X.509) | (n/a) | (n/a) | **Key-pair JWT** | App private key (1 h tokens) |
| No-secret federation | **Federated identity credential** | `AssumeRoleWithWebIdentity` + OIDC provider | **Workload Identity Federation** | **OAuth token federation** | **Workload identity federation** | OIDC issuer for Actions |
| Short-lived token service | Entra token endpoint | **STS** | **STS** / IAM Credentials API | `/oidc/v1/token` | OAuth / session tokens | Installation tokens |
| Permission model | Azure RBAC + Entra roles + app permissions | IAM policies, SCPs, boundaries | IAM roles on resources, org policies | Unity Catalog grants + ACLs | Roles + policies | Repo/org permissions |
| Just-in-time admin | **PIM** | Identity Center + temporary elevated access | **PAM** | (via IdP groups) | (via IdP / SCIM groups) | (via IdP) |
| Vault | **Key Vault** | **Secrets Manager** + KMS | **Secret Manager** + Cloud KMS | Secret scopes | `SECRET` objects | Actions secrets |
| Audit log | Sign-in + audit logs, Activity Log | **CloudTrail** | **Cloud Audit Logs** | `system.access.audit` | `ACCESS_HISTORY`, `LOGIN_HISTORY` | Audit log |

### 14.2 Decision guide

&#91;embedded content: Identity and credential decision tree · 4 questions, 5 outcomes\]

Stop at the first "yes". The two highlighted outcomes leave nothing to steal; every step further down adds a secret you must store, watch and rotate.

### 14.3 Quick answers to common questions

| Question | Answer |
| --- | --- |
| Managed identity or service principal? | MI when the code runs **in Azure**; otherwise an SP with a **federated credential**, or a certificate if federation isn't possible |
| Client secret or certificate? | Certificate (or federation). Secret only for short tests or tools that support nothing else |
| Is the Client ID secret? | No. It's like a username. The **secret, certificate or token** is what you protect |
| Where do I store a secret I can't avoid? | In the platform's vault, read at runtime by the workload's identity; expiry and owner set |
| Access token or API key for my own API? | Entra/OIDC tokens for internal and partner apps (identity + expiry built in); API keys behind a gateway for simple metered public access |
| Service account key for a vendor? | Grant the **vendor's own** SA or role (or use External ID / federation). Only create a key as a last resort |
| PAT for automation? | No. Use a service principal (Databricks), service user + key pair (Snowflake), or GitHub App (GitHub) |
| User delegated or app-only permissions? | Delegated when acting for a signed-in user (never more than the user may do); app-only for daemons, with minimal scopes |

## Module 15 — Best practices, rotation, incident response, quiz and cheat sheet

### 15.1 Ten rules that prevent most incidents

1. **No secret is better than a vaulted secret:** managed identity / attached SA / IAM role → federation → certificate → vaulted secret.
2. **One identity per workload, per environment.** Never share an SP or SA between dev and prod, or between two apps.
3. **Groups for humans; no standing admin;** PIM, PAM or Identity Center for elevation.
4. **Phishing-resistant MFA** (FIDO2 / passkeys) for all admins; MFA for everyone.
5. **Least privilege at every layer:** resource RBAC → object grants → RLS → masking.
6. **Short lifetimes:** tokens 1 h, secrets ≤ 6 months, PATs ≤ 30–90 days, SAS in hours.
7. **Every credential has an owner and an expiry alert.**
8. **Scan for leaks:** GitHub secret scanning + push protection, pre-commit hooks (`gitleaks`), and cloud leak-detection programmes (AWS and GitHub partners auto-notify).
9. **Log centrally and alert:** sign-ins of workload identities from new IPs, secret reads outside expected identities, role grants outside CI.
10. **Rehearse the leak drill** below twice a year.

### 15.2 Rotation playbook (zero downtime)

| Credential | Rotation steps |
| --- | --- |
| Entra client secret | Add new secret → update vault → apps pick it up → verify sign-ins use the new key ID → delete old secret |
| Certificate | Key Vault auto-renew → upload new public key to the app (`--append`) → switch → remove old |
| AWS access key | Create second key → deploy → check `LastUsed` on the old key → deactivate → delete after 7 days |
| GCP SA key | Same two-key pattern; better, delete keys and move to WIF |
| Snowflake key pair | `RSA_PUBLIC_KEY_2` → switch client → unset `RSA_PUBLIC_KEY` |
| Databricks OAuth secret | Create second secret on the SP → update consumers → delete old |
| DB password in Secrets Manager | Automatic rotation (Lambda) or IAM database auth |

### 15.3 Incident response: "a secret was leaked"

1. **Revoke first, investigate second.** Disable or delete the secret/key/token immediately (`az ad app credential delete`, `aws iam update-access-key --status Inactive`, `gcloud iam service-accounts keys disable`, `ALTER USER ... UNSET RSA_PUBLIC_KEY`).
2. **Kill sessions:** revoke refresh tokens and sessions, and disable the identity if needed.
3. **Scope the blast radius:** list what the identity could access (role assignments, policies, grants).
4. **Hunt in logs** from the moment of exposure: Entra sign-in logs (service principal sign-ins), CloudTrail, Cloud Audit Logs, `system.access.audit`, `LOGIN_HISTORY`. Look for unfamiliar IPs, new resources, data exports and role grants.
5. **Remove persistence:** check for new users, SPs, keys, federated credentials, role assignments and access keys the attacker may have created.
6. **Replace** the credential with a stronger type (federation, MI), not just a new copy of the same secret.
7. **Clean the source:** remove it from Git history (and still treat it as compromised), chat logs and images.
8. **Review billing** for crypto-mining or large egress, and contact the cloud provider if suspension or abuse is involved.
9. **Write the post-mortem:** how it leaked, why detection took that long, and what control prevents a repeat.

### 15.4 Anti-patterns to recognise instantly

| You see | Why it's bad | Replace with |
| --- | --- | --- |
| `client_secret = "Abc8Q~..."` in a notebook | Anyone with notebook access, or Git history, has it | Managed identity / secret scope / Key Vault reference |
| Pipeline "run as" an engineer | Breaks on departure; over-privileged; wrong audit actor | Service principal or managed identity |
| `Owner` / `Editor` / `ACCOUNTADMIN` for an app | Total compromise if leaked | Narrow built-in or custom role |
| SA JSON key emailed to a vendor | Permanent, untraceable copies | Grant the vendor's own identity, or WIF |
| Storage account key in config | Full control of all data, can't be scoped | Data-plane RBAC + disable shared key |
| Federation trust on `repo:org/*` | Any repo or branch becomes prod | Exact repo + branch or environment |
| Report users as workspace Contributors | RLS silently bypassed | Viewer or app audience |

### 15.5 Knowledge check

1. What's the difference between an app registration and a service principal?
2. When can you **not** use a managed identity, and what do you use instead?
3. Which is secret: Tenant ID, Client ID, client secret, or certificate thumbprint?
4. A call returns 403. Is the problem authentication or authorization?
5. What does PKCE protect against?
6. Why should a SPA never contain a client secret?
7. Which JWT claim tells an API whether the token was meant for it?
8. Why is a user delegation SAS safer than an account SAS?
9. In AWS, what does an External ID in a role trust policy prevent?
10. What does `roles/iam.serviceAccountTokenCreator` on a service account allow?
11. Why does RLS in a Power BI model not apply to a workspace Contributor?
12. Give the four steps to rotate an Entra client secret without downtime.

**Answers**

1. The app registration is the global definition (Client ID, credentials, permissions). The service principal is the app's identity in a specific tenant, which receives role assignments.
2. When the code doesn't run in Azure (GitHub Actions, AWS, GCP, on-premises, laptops). Use workload identity federation, or a certificate if federation isn't supported.
3. Only the client secret (and the certificate's private key). IDs and thumbprints are identifiers.
4. Authorization. The identity was recognised but lacks permission.
5. Interception of the authorization code. Without the original `code_verifier`, a stolen code can't be exchanged for tokens.
6. Anything shipped to a browser is public, so the secret would be exposed to every user. SPAs are public clients and use PKCE without a secret.
7. `aud` (audience).
8. It's signed with an Entra-issued key tied to an identity, can be scoped and revoked without rotating the storage account key, and is auditable per identity.
9. The confused-deputy problem: another customer of the same vendor tricking the vendor into assuming *your* role.
10. The holder can impersonate that SA, minting its access and ID tokens, so they effectively have all its permissions.
11. Workspace Admin, Member and Contributor have write access to the dataset, and RLS applies only to read-only users (Viewers, app users, build-only users).
12. Add a new secret → update the vault and consumers → verify the new key is in use (sign-in logs) → delete the old secret.

### 15.6 One-page cheat sheet

| If you need… | Use |
| --- | --- |
| Code in Azure to call Azure, Fabric or SQL | **Managed identity** + `DefaultAzureCredential` |
| Code outside Azure to call Azure or Fabric | **SP + federated credential**, else certificate |
| Code in GCP | **Attached service account** + ADC |
| Code in AWS | **IAM role** (Lambda, task, instance profile) |
| CI/CD to any cloud | **OIDC federation** (GitHub/Azure DevOps → Entra / AWS / GCP) |
| A person on a server without a browser | Device code flow (restricted by CA) |
| App signing users in | OIDC authorization code + PKCE |
| Daemon calling an API | OAuth client credentials |
| API calling another API as the user | On-behalf-of |
| External partner without your identity | B2B guest, or per-partner SP / service user, or a short SAS |
| Databricks automation | Service principal + OAuth M2M / token federation |
| Snowflake automation | `TYPE=SERVICE` user + key pair / WIF |
| GitHub automation | GitHub App or `GITHUB_TOKEN` |
| Any secret you can't avoid | Platform vault + expiry + owner + rotation |
| Restrict rows | RLS / row access policy with a mapping table |
| Hide columns or tables | OLS / CLS / policy tags; masking for display |
| Admin rights | PIM / PAM / Identity Center, time-bound |

### Further reading

These are official documentation hubs; product names and portal labels change often, so check them against your tenant. The links weren't individually opened for this draft.

- [Microsoft identity platform documentation](https://learn.microsoft.com/entra/identity-platform/)
- [Managed identities for Azure resources](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)
- [Workload identity federation (Entra)](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Google Cloud service accounts](https://cloud.google.com/iam/docs/service-account-overview)
- [Google Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [AWS IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [OAuth 2.0 (RFC 6749)](https://datatracker.ietf.org/doc/html/rfc6749) and [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)
- [GitHub Actions OIDC](https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
