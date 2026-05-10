# Phase 1 — Azure provisioning (initial) — step by step

**Audience:** You are in **Spain**; this guide picks **EU regions** and **English Azure portal** names (UI may show Spanish labels — map by resource type).  
**Design source:** [POC-Azure-AI-200-task-and-design.md](POC-Azure-AI-200-task-and-design.md) (locked decisions **§8**).  
**App runtime:** The API is built as a **.NET 10** container image (see [Visual Studio Phase 1 doc](POC-Azure-AI-200-phase1-visual-studio.md)); Container Apps only needs the correct **ingress port** and image reference.  
**Scope:** **Phase 1 only** — no Service Bus, Functions, API Management, or Entra for this milestone.

---

## 1. Naming convention (replace placeholders)

Use **one resource group** for everything in Phase 1. Pick a **suffix** that is unique to you (initials + digits), **lowercase**, no spaces.

| Item | Example pattern | Notes |
|------|-----------------|--------|
| **Resource group** | `rg-ai200-poc-es-abc1` | Holds all Phase 1 resources. **Teardown** = delete this group (see main doc **§8 D3**). |
| **Region** | `spaincentral` **or** `westeurope` | **Spain Central** (Madrid) for Spain proximity; **West Europe** (Netherlands) if a service SKU is unavailable in Spain Central. Stay in **one** region per POC. |
| **Container Registry** | `acrai200esabc1` | **Globally unique**, 5–50 alphanumeric. **SKU:** `Basic` for POC cost (no geo-replication). |
| **Container Apps environment** | `cae-ai200-es-abc1` | Logical boundary for ACA apps + logs. |
| **Container App (API)** | `ca-api-ai200-abc1` | Your HTTP API workload. |
| **Cosmos DB account** | `cosmos-ai200-es-abc1` | Globally unique. API **NoSQL**. |
| **Storage account (Blob)** | `stai200esabc1` | **Globally unique**, 3–24 lowercase letters/numbers only. |
| **Key Vault** | `kv-ai200-es-abc1` | **Globally unique**, max 24 chars. |
| **Log Analytics workspace** | `log-ai200-es-abc1` | Feeds KQL + Application Insights. |
| **Application Insights** | `appi-ai200-es-abc1` | Link to workspace above. |
| **Azure OpenAI** (if used) | `oai-ai200-es-abc1` | Name + region subject to model availability — check portal quotas. |

**Budget:** In **Cost Management + Billing** → **Budgets**, create an alert (e.g. **€35/month** or your cap) scoped to this resource group.

---

## 2. Region choice (Spain)

1. Prefer **Spain Central** (`spaincentral`) if **Cosmos DB for NoSQL**, **Container Apps**, **Azure OpenAI**, and **embedding model** availability match your subscription offer.  
2. If any resource type is **greyed out** or missing models, use **West Europe** (`westeurope`) for **all** resources in the same group (avoid cross-region latency and private-link complexity for a first POC).

---

## 3. Provisioning order (portal — recommended for first pass)

Complete steps **in order** so dependencies exist before linking (names from §1).

### Step A — Resource group

1. Portal → **Resource groups** → **Create**.  
2. **Subscription:** yours. **Region:** Spain Central or West Europe. **Name:** `rg-ai200-poc-es-{suffix}`.  
3. **Create**.

### Step B — Log Analytics + Application Insights

1. **Create a resource** → **Log Analytics workspace** → **Region** = same as RG → name `log-ai200-es-{suffix}`.  
2. **Create a resource** → **Application Insights** → link to the **workspace** → name `appi-ai200-es-{suffix}`.  
3. Note the **Instrumentation Key** / **Connection String** (you will move secrets to Key Vault later; for first smoke test you may use Key Vault or ACA secrets).

### Step C — Azure Container Registry

1. **Container Registry** → **Create**. **SKU:** Basic. **Location** = same region. **Name:** `acrai200es{suffix}` (adjust length).  
2. Enable **Admin user** only if you need quick CLI login for class demos; prefer **Managed identity** pull from ACA in production-style setups.  
3. **Create**.

### Step D — Azure Cosmos DB for NoSQL

1. **Azure Cosmos DB** → **Create** → API **NoSQL** (formerly Core).  
2. **Capacity mode:** for cost experiments use **Serverless** *if available in region*; otherwise **provisioned throughput** with minimal RU/s (validate current portal pricing).  
3. After creation: create a **database** (e.g. `ragdb`) and a **container** (e.g. `chunks`) with a **partition key** such as `/documentId` (or `/tenantId` if multi-tenant later).  
4. Configure **vector indexing and embedding path** per current Microsoft Learn for Cosmos DB for NoSQL vector search (dimensions must match your embedding model, e.g. 1536).

### Step E — Storage account (Blob)

1. **Storage account** → **Standard** performance, **Locally-redundant storage (LRS)** for POC. **Kind:** StorageV2.  
2. **Networking:** for learning, “Enable public access from all networks” is simplest; tighten later.  
3. Create a **container** (e.g. `corpus`) access type **Private** (API uses keys or RBAC).  
4. **Soft delete** for blobs: note retention — affects **cost after teardown** if you delete the RG with soft-delete retention.

### Step F — Azure Key Vault

1. **Key Vault** → **Create** → same region → name `kv-ai200-es-{suffix}`.  
2. **Pricing:** Standard.  
3. After deploy: you will add **secrets** (Cosmos connection string or reference, OpenAI key, storage connection string) and grant the **Container App managed identity** **Get** access via **Access policies** or **RBAC** (Key Vault Secrets User).

### Step G — Azure OpenAI or Azure AI Foundry

1. **Azure OpenAI** (or Foundry project per your org policy) in the **same** or **paired** region per documentation (OpenAI has regional model tables).  
2. Deploy an **embedding** model (e.g. `text-embedding-3-small` or region-supported equivalent).  
3. Store **endpoint + key** in Key Vault (or use **Microsoft Entra** auth to OpenAI if you adopt token-based access later — optional complexity).

### Step H — Container Apps environment + app (placeholder)

1. **Container Apps** → **Create** → **Create Container Apps environment** — name `cae-ai200-es-{suffix}`.  
2. Link **Log Analytics** to the environment (observability).  
3. Create the **Container app** `ca-api-ai200-{suffix}` with a **public ingress** (HTTPS), **port** matching your ASP.NET Core app (typically **8080** or **80** depending on Dockerfile `ASPNETCORE_URLS`).  
4. For **first deploy**, you can use a **hello-world** image from Microsoft Samples, then swap to your image from ACR once CI/local push is ready.  
5. Assign **System-assigned managed identity** to the app; grant it **data plane** roles on **Cosmos**, **Storage**, **Key Vault** as you implement (Cosmos DB Built-in Data Contributor, Storage Blob Data Contributor, Key Vault Secrets User — exact role names depend on RBAC model; validate in portal).

### Step I — (Optional learning) Container Instances

Not required for the main API (**§8 D1**). If you want the drill: after pushing an image to ACR, use **Azure CLI** from Cloud Shell: `az container create` referencing the image — see [Container Instances quickstart](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-quickstart).

---

## 4. After provisioning — sanity checklist

- [ ] All resources in **one** region (except OpenAI if policy forces another — then document exception).  
- [ ] **ACA** health probe passes once your API image is deployed.  
- [ ] **Cosmos** container accepts a test **upsert** with a dummy vector dimension.  
- [ ] **Blob** upload works (portal **Upload** or Storage Explorer).  
- [ ] **Key Vault** secret read works from local dev (your user) or from ACA (managed identity).  
- [ ] **Application Insights** shows **requests** when you hit the API.

---

## 5. Teardown (cost control — **§8 D3**)

When you pause the POC: **Resource groups** → select `rg-ai200-poc-es-{suffix}` → **Delete resource group**. Confirm you no longer need data in Cosmos/Blob. If **soft delete** is on for blobs or Cosmos backups exist, review billing docs so you are not surprised by retention charges.

---

## 6. Next step

Create the code solution: [POC-Azure-AI-200-phase1-visual-studio.md](POC-Azure-AI-200-phase1-visual-studio.md).

---

## 7. Phase 2 preview (do not create yet)

**Azure Service Bus** namespace + **Azure Functions** app — follow the main design doc after Phase 1 works.
