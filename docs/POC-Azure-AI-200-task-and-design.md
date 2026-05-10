# Proof of concept — AI-200 exam-aligned design (containers + Cosmos)

**Purpose:** Hands-on practice aligned with **[Exam AI-200](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)** skills (containers, Cosmos DB vectors, messaging, Functions, security, observability), cost-conscious (~**$35/month** budget), teardown-friendly. **Phase 1** focuses on **semantic search** (sync ingest + query); **Phase 2** adds **async** messaging and Functions for exam breadth.  
**Status:** Design **locked** for Phase 1–2 scope (see **§8**); use companion docs for provisioning and Visual Studio setup.  
**Last reviewed:** 2026-05-09  

**Folder naming:** The repo may live under `Prueba Concepto AZ-200` locally for your organization. The **certification / exam** this POC targets is **AI-200** (Developing AI Cloud Solutions on Azure). The legacy exam **AZ-200** on Learn is an **old, retired** developer exam (2019); do not confuse the folder label with that credential.

---

## 1. Exam alignment (verified skills outline)

**Source:** [Study guide for Exam AI-200](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200) (Microsoft Learn).

**Why Cosmos DB for this POC (not PostgreSQL):** The AI-200 guide lists **Azure Cosmos DB for NoSQL** with **embeddings and vector similarity search** as explicit skills. **Azure Database for PostgreSQL** (including pgvector) also appears on the exam — but **Cosmos DB is named in the exam’s study resources link list** (PostgreSQL is not in that particular list). For **one** primary data plane in this POC, **Cosmos DB for NoSQL** is the clearer choice to stay aligned with both the skills bullets and the published documentation index.

**Domains (summary):**

| Domain | Focus |
|--------|--------|
| **Containerized solutions (20–25%)** | ACR, Container Apps, AKS, App Service, KEDA |
| **AI + data (25–30%)** | Cosmos DB (vectors, change feed), PostgreSQL, Azure Managed Redis |
| **Connect / consume (20–25%)** | Service Bus, Event Grid, Azure Functions |
| **Secure / monitor (20–25%)** | Key Vault, App Configuration, OpenTelemetry, KQL |

**Azure AI Foundry / model APIs:** Not named as a skill in the AI-200 outline. Use them **as supporting components** (e.g. **embedding** or **chat completion** for RAG). The core exam practice here is **Azure platform integration**, not catalog Vision/Speech demos alone. For **semantic search vs RAG**, **ingest / index / query**, and **Phase 1 vs Phase 2**, see **§2**.

---

## 2. Concept — what we are building

### 2.1 Elevator pitch (Phase 1 — semantic search)

You build a **semantic search** system over **your own text**: **`.txt` (or `.md`) files** are stored in **Azure Blob Storage**, the app **splits them into chunks**, calls an **embedding model** (via **Azure OpenAI** or **Azure AI Foundry**) to turn each chunk into a **vector**, and stores those vectors (plus text metadata) in **Azure Cosmos DB for NoSQL** so you can run **vector similarity search**. Users call a **containerized HTTP API**: you **package the API with Docker**, push the image to **Azure Container Registry**, and run it on **Azure Container Apps** (**locked** host — §8: scaling, revisions, **KEDA** for exam alignment).

**Optional later:** add a **chat completion** call so retrieved passages feed an LLM (**RAG**). **Phase 2 (separate iteration):** **Azure Functions** + **Azure Service Bus** for **async** ingest (locked choice — see §8) — not part of the first milestone (see §4).

**Utility (why it is interesting to program):** The API answers *“which passages in **my** corpus are **meaningfully** closest to this question?”* — not *“which rows contain this substring?”* like SQL `LIKE`. Example: query *“refund policy”* can rank a chunk that only says *“money-back within 30 days”* highly, even without shared keywords.

**C#:** Primary implementation language in Visual Studio; keep **Python** awareness for exam wording (candidate profile lists Python).

### 2.2 What the pitch means — three layers

| Layer | What happens | Phase 1 (this doc’s default) |
|--------|----------------|------------------------------|
| **Ingest** | Text lands in **Blob**; the app reads it, **splits into chunks** (paragraphs / token windows), assigns ids (file + chunk index). | Same API or ops workflow uploads `.txt` to Blob; **indexing runs in the API request** (sync) or a **manual “reindex” call** — no queue yet. |
| **Index** | For each chunk: call **embedding** API → get vector → **upsert** document in **Cosmos** (chunk text, blob path, **embedding** field for vector search). This is **indexing** (building the searchable vector representation). | Cosmos **vector index** + items per chunk. |
| **Serve** | User sends a **natural-language query**; app embeds the query, runs **vector similarity search** in Cosmos, returns **top K** chunks (e.g. **top 3 passages**) with **similarity scores**. | No `LIKE` on the paragraph text for the core match — the match is **vector distance** in Cosmos (see §2.6). |

You do **not** “traverse a list” in your own code for search at query time: **Cosmos DB** (with a vector index) returns the nearest neighbors. You *do* traverse/read blobs **once** at ingest to produce chunks and embeddings.

### 2.3 Semantic search vs SQL `LIKE` (and full-text)

| Approach | What it matches | Good for |
|----------|------------------|----------|
| **SQL `LIKE '%word%'`** | Character patterns / keywords | Exact or partial string matches |
| **Semantic search (this POC)** | **Meaning** via embeddings | Questions, paraphrases, concepts without the same words |

Full-text search is a middle ground (words, stems, sometimes synonyms). **Semantic search** uses **vectors**: similar meaning → **geometrically close** vectors → high **similarity score**.

### 2.4 Embeddings (concept) and **chunks**

Models do not read a whole book as one embedding in practice: they work on **bounded spans of text** called **chunks**.

**What is a chunk?** A **chunk** is one **segment** of your source document after **splitting** — for example a **paragraph**, a **fixed number of tokens** (e.g. 256–512), or a **sliding window** with a little overlap so sentences at boundaries are not cut in half blindly. Each chunk becomes **one row (item)** in Cosmos: same metadata (which file, chunk index), the **raw chunk text**, and **one embedding vector** for that chunk. **Why chunk?** Embedding models have a **maximum input length**; long files must be split. **Smaller chunks** give finer search hits but more items and more embedding API calls; **larger chunks** carry more context per vector but may mix unrelated topics. For this POC, start with **paragraph or simple fixed-size splits** and tune later.

An **embedding** is a **fixed-length array of floats** produced by a dedicated **embedding model** (exposed as an HTTP API in Azure OpenAI / Foundry). It is a **numeric fingerprint of the meaning** of **that chunk** (not usually a single isolated word — words inside the chunk contribute to one vector for the whole segment).

> **Note (ES — embedding):** Un embedding es un array de números decimales (floats) de longitud fija generado por un modelo. Es una “huella numérica” comprimida que representa el significado o la esencia de un texto. **La analogía:** es como una función hash, pero con una diferencia clave: en el hashing criptográfico, si cambias una sola letra, el resultado es totalmente distinto. En los embeddings, si dos textos tienen un significado similar, sus vectores resultantes estarán cerca el uno del otro en el espacio matemático.

> **Note (ES — chunk / trozo):** Un **chunk** (trozo o fragmento) es un **segmento de texto** obtenido al **dividir** un archivo largo en partes más pequeñas (por párrafos, por número máximo de tokens, etc.). **No** se genera normalmente un embedding por cada palabra suelta: se genera **un vector por chunk**, de modo que el modelo resume el **significado de ese fragmento completo**. Así puedes buscar “en qué parte del documento” está la idea, no solo coincidencias de palabras sueltas.

**Who generates embeddings?** Always the **embedding model endpoint** — not “RAG” and not the vector DB. **RAG** only describes what you do **after** retrieval (optional LLM step). **Indexing** = chunk text → **embedding API** → store vector in Cosmos.

### 2.5 What you ingest (concrete) — `.txt` in Blob

- **Corpus:** One or many **plain-text files** (policies, FAQs, notes, sample docs).  
- **Unit stored in Cosmos:** **Chunks** (e.g. 200–500 tokens or paragraph splits), each with: `id`, `sourceBlobPath`, `chunkIndex`, `text`, `embedding` (vector).  
- **Is it “words”?** Not one Cosmos item per random word. You **index meaningful segments** (chunks). Words appear inside chunk text; the vector represents the **whole chunk’s** meaning.  
- **Putting files in Blob (Phase 1):** Any of: **HTTP upload** through your API (multipart or “register URL”), **Azure Storage Explorer**, **AzCopy**, or CI drop — whatever is simplest. **Trigger for indexing (Phase 1):** e.g. `POST /index` or `POST /documents/{id}/reindex` that reads Blob, chunks, embeds, writes Cosmos **in the same flow** (sync). No Service Bus requirement.  
- **Phase 2:** Blob upload → **Service Bus** message → **Azure Function** worker (locked pattern — see §8).

### 2.6 Query time — how search runs (semantic only vs RAG)

**Semantic search only (Phase 1 default):**

1. User query string → **embedding API** → query vector.  
2. **Cosmos DB vector similarity search** (e.g. top **3**) using that vector — this is the “search engine”; it is **not** SQL `LIKE` on paragraphs for ranking.  
3. API returns those **passages** + **similarity / distance scores** (Cosmos returns a score or distance per hit — use it to label “high / medium / low” confidence or to filter weak matches).

**If nothing is truly similar:** scores stay **low** or distances **high**. You can define thresholds (e.g. discard hits below a cutoff and return *“no good match”*) or still return top 3 but flag them as low confidence using **three levels** (e.g. strong / uncertain / weak from score bands).

**RAG (optional extension):** Steps 1–2 are **identical**. Then you send **retrieved chunk text + user question** to a **chat completion** LLM; the LLM **writes the answer** in natural language. The LLM does **not** replace Cosmos for finding passages — **retrieval is still vector search in Cosmos**; the LLM only **generates** text from what you retrieved.

**Clarifying common doubts:**

- **Without RAG, is search “a query on Cosmos”?** Yes: **vector search** in Cosmos (similarity), not keyword `LIKE`.  
- **With RAG, does the LLM run the search?** **No.** Your app runs **embedding + Cosmos vector query**; the LLM only **reads** the snippets you pass it and **composes** an answer.

### 2.7 Other uses of embeddings + vector store (beyond “find similar passage”)

Same building blocks support: **semantic deduplication**, **clustering** similar chunks, **classification** (embed labels and pick nearest class), **anomaly** detection (far from all centroids), **recommendations** (“similar to this chunk”). This POC stays focused on **search + top‑K passages**.

### 2.8 Why this fits AI-200

Touches **containers**, **Cosmos vectors**, **Blob**, **Key Vault**, **monitoring** in Phase 1; **Functions**, **messaging**, **Event Grid**, **Redis**, etc. in later stretches — aligned with the [AI-200 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200) domains.

### 2.9 Phase 1 glossary — concepts this POC exercises

Short definitions of **every major Phase 1 building block** (beyond §2.2–2.7). Use this as a study map; details stay in Microsoft Learn ([AI-200 study resources](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200#study-resources)).

#### Retrieval-augmented generation (RAG)

**RAG** is a **pattern**, not a database or a single Azure product. After **retrieval** (vector search in Cosmos returns the best text chunks), you optionally call a **large language model (LLM)** with: **(user question + retrieved passages as context)**. The model **generates** a natural-language answer **grounded** in your data. **Retrieval** is still **your code + Cosmos vector search**; the LLM does **not** replace that step. Phase 1 can ship **without** RAG (return passages only); Phase 1b adds the completion call.

#### Vector database vs “a database that supports vectors”

A **vector database** is often a product **optimized** for storing billions of vectors and nearest-neighbor search (many are standalone services). In this POC, **Azure Cosmos DB for NoSQL** is a **general-purpose document database** that also supports **vector indexing and similarity search** — exam-relevant and enough for a corpus of chunks. Conceptually you still treat it as your **“vector store”** for embeddings.

#### Azure Cosmos DB for NoSQL (in this POC)

**Cosmos DB for NoSQL** stores **JSON items** (your chunk documents: text, metadata, **embedding** array). You define a **container**, **partition key** (e.g. `documentId` or `tenantId`), **indexing policy**, and a **vector index** so queries can request **“top K items closest to this query vector”**. You pay attention to **Request Units (RUs)** and consistency — both appear in AI-200 skills. The app uses the **Cosmos SDK** from the API container.

#### Azure Blob Storage

**Blob Storage** holds **raw files** (here, `.txt`/`.md`). It is **cheap, durable object storage** — not a query engine for meaning. The API **reads** blobs during **ingest/index**, then persists **searchable** data in Cosmos.

#### Azure OpenAI / Azure AI Foundry (embedding, optional chat)

**Azure OpenAI** and **Azure AI Foundry** expose **HTTP APIs** for models. For Phase 1 you need an **embedding** deployment (turns text → vector). For RAG you add a **chat completion** deployment. Credentials are secrets — see Key Vault.

#### Azure Key Vault

**Key Vault** is the **central place for secrets** (API keys, connection strings, Cosmos keys if not using RBAC-only paths). The container app uses **Managed Identity** so the API can **read** secrets at runtime **without** embedding keys in the image or git. AI-200 explicitly covers **Key Vault** and secret handling.

#### Monitoring, Application Insights, Log Analytics, KQL

**Monitoring** here means **metrics + logs + distributed traces** so you can see latency, failures, and dependencies (API → Cosmos → OpenAI). **Application Insights** is an **Azure Monitor** feature: your app sends **telemetry** (requests, dependencies, exceptions). Data lands in a **Log Analytics workspace**, where you write **KQL** queries to analyze logs — an AI-200 skill. **OpenTelemetry** (OTel) is an optional library path to emit the same kinds of traces.

#### Docker and the container image

**Docker** packages your **ASP.NET Core** app plus runtime into an **image** (immutable artifact). A **Dockerfile** describes **build** steps (`dotnet publish`, base image, entrypoint). You run **containers** from that image locally (`docker run`) or in Azure. Images are **not** VMs: they share the host kernel, start fast, and carry **only** what the app needs.

#### Azure Container Registry (ACR)

**ACR** is your **private Docker registry** in Azure. You **push** images (`docker push`), tag versions, optionally **build in the cloud** (ACR Tasks — exam topic). Container Apps (or ACI) **pull** from ACR. **Learning ACR in this POC is essential** — it is the standard path from Visual Studio / CI to a deployed container.

#### Azure Container Apps (ACA) — preferred runtime for this POC

**Container Apps** runs **containers** on a **managed environment**: **HTTPS ingress**, **revisions** (blue/green style), **environment variables** and **secrets**, and **horizontal scale** (add/remove replicas of your container). It is a good place to **learn containers on Azure** beyond “I already know App Service”: you think in **images**, **ports**, **probes**, and **identity** attached to the app. This POC **locks** hosting to **ACA** (see §8).

#### KEDA (Kubernetes Event-driven Autoscaling)

**KEDA** is an **open-source** component that **scales** workloads (including **Azure Container Apps** and **AKS**) based on **external signals** — not only CPU/memory. Examples: **length of a Service Bus queue**, **Azure Storage queue depth**, **Kafka lag**, **Cron** schedule. When the queue is empty, KEDA can scale **in to zero** (or to a configured minimum); when messages arrive, it scales **out** so your app processes backlog faster. That is why AI-200 pairs **Container Apps** with **KEDA**: event-driven **serverless-style** scaling for containers. In **Phase 1** this POC may run at **fixed min replicas**; **Phase 2** (Service Bus + Functions or ACA consuming the bus) is where **queue-based KEDA** becomes a natural exam-aligned story.

#### Azure Container Instances (ACI) — useful to learn, optional for the main path

**Container Instances** runs **one or a few containers** without orchestrating a full **Container Apps** environment — minimal **“run this image with these env vars”** ([Container Instances documentation](https://learn.microsoft.com/en-us/azure/container-instances/)). It appears in the [AI-200 “Find documentation” list](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200#study-resources), so it is **worth a small side exercise**: e.g. pull the **same** image you pushed to ACR and run it with `az container create` to see **pull → run → public IP** in minutes. **For the full semantic-search API** (stable URL, easy TLS, scaling, revisions), **Container Apps** is still the **primary** target — ACI complements learning; it does not replace ACA for the main production-like story in this design.

**Summary:** **ACR + Docker + Container Apps** = core Phase 1 container skills. **ACI** = quick exam-aligned drill on “run a container from ACR” if you want extra practice.

---

## 3. Target architecture (conceptual)

**Phase 1:** The diagram below includes **solid** lines for the **sync** path (API ↔ Blob, Cosmos, embedding endpoint, Key Vault, telemetry). **Dashed** components are **Phase 2+** (async, extra exam stretches).

```
[Client / Postman]
        |
        v
[Container Apps – hosts ASP.NET Core API container]  <── Phase 1 (locked — §8)
        ^
        | image from [Azure Container Registry]
        |
        +--> [Azure Cosmos DB for NoSQL] — documents/chunks metadata + vector embeddings
        |
        +--> [Blob Storage] — .txt / .md corpus uploads (Phase 1)
        |
        +--> [Azure OpenAI / Foundry] — embedding model (Phase 1); optional chat for RAG
        |
        +--> [Azure Key Vault] + Managed Identity where possible
        |
        |    .  .  .  Phase 2+ (async / optional exam stretches)  .  .  .
        +~~> [Azure Functions] — consume Service Bus; embed; write Cosmos
        +~~> [Azure Service Bus] — queue decouple ingest (locked — §8)
        +~~> [Event Grid – OPTIONAL] — e.g. blob-created → enqueue / Function
        +~~> [Azure Managed Redis – OPTIONAL] — cache hot queries
        |
        +--> Application Insights / OpenTelemetry + Log Analytics workspace — metrics, traces, **KQL**
```

**Deployment path (Phase 1):** Develop **ASP.NET Core** locally → **Dockerfile** → build image → **push to ACR** → deploy to **Azure Container Apps** (§8 D1). Optionally run the same image on **Azure Container Instances** once as a **learning drill** (not the main API host).

**See also:** §2.9 (Docker, ACR, ACA, ACI, Key Vault, monitoring).

### 3.1 Architecture ↔ AI-200 skills goals (POC mapping)

The exam name is **AI-200** (not the retired legacy **AZ-200** label some folders use). This table maps **this POC’s** components to **skills you rehearse**.

| POC component | AI-200 domain (see [study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)) |
|---------------|----------------------------------------------------------------------------------------------------------------------------------|
| **ACR** + **Container Apps** (locked) + optional **ACI** drill (learning only) | Develop containerized solutions — build, deploy, env config |
| **Cosmos DB for NoSQL** + **vector search** + items/chunks | Develop AI solutions — embeddings, similarity search, SDK queries, indexing policies / RU awareness |
| **Blob Storage** | Azure data / ingest patterns (exam data-management context) |
| **Azure OpenAI / Foundry** (embedding; optional chat) | Consuming Azure AI **as a dependency** of your solution (supporting; exam emphasis remains platform skills) |
| **Key Vault** + **Managed Identity** | Secure solutions — secrets |
| **Application Insights** + **KQL** | Monitor and troubleshoot |
| **Service Bus** + **Functions** (Phase 2, locked) | Connect and consume — messaging + serverless |
| **Event Grid**, **Redis** | Optional later stretches (this POC does **not** include API Management or Entra — §8) |

**Study resources (Find documentation):** [Azure documentation](https://learn.microsoft.com/en-us/azure/?product=featured) and the service links in **§7** mirror the [AI-200 “Find documentation” table](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200#study-resources).

---

## 4. Scope tiers — Phase 1 first, then stretches

| Tier | Included | Exam relevance |
|------|------------|----------------|
| **Phase 1 (must)** | **Docker** + **Dockerfile**; **ACR** push; **Azure Container Apps** for API (**§8**); optional **ACI** one-off drill; **Cosmos DB** (vector + vector query); **Blob** (`.txt`/`.md`); **sync** ingest/index; **embedding** calls to **Azure OpenAI** / **Foundry**; **semantic search** API (top‑K + scores); **Key Vault** + **Managed Identity**; **Application Insights** / **Log Analytics** + **KQL** | Containers + Cosmos + Blob + AI endpoint consumption + secrets + observability |
| **Phase 1b (optional)** | **Chat completion** step after retrieval (**RAG** answer) | Same platform; adds LLM orchestration practice |
| **Phase 2 — Stretch A (locked messaging)** | **Azure Function** triggered by **Azure Service Bus** (queue or topic/subscription) after upload / event | Functions + Service Bus (explicit exam bullets) |
| **Stretch B** | **Event Grid** subscription (e.g. BlobCreated → Function or enqueue) | Event-driven workflows |
| **Stretch C** | **Azure Managed Redis** for caching query results or embeddings lookup | Redis bullet |

**Out of POC scope (locked):** **API Management** and **Entra ID** are **not** part of this project; study them separately via [Learn](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200#study-resources) if needed for the exam.

**Avoid scope creep:** Finish **Phase 1** before **Service Bus + Functions**. Do **not** add **Event Hubs** or multiple optional services at once. Pick **one** optional stretch (**B** or **C**) per iteration after Phase 2A.

**PostgreSQL:** Out of scope for this POC’s **primary** store (Cosmos is the focus). Revisit only if you want a second project later.

---

## 5. Visual Studio — project shape

| Artifact | Role |
|----------|------|
| **ASP.NET Core Web API** (.NET 10) | Main HTTP surface; Swagger for tests (see [Phase 1 — Visual Studio](POC-Azure-AI-200-phase1-visual-studio.md)) |
| **Dockerfile** + optional **docker-compose** (local only) | Same image as production path to ACR |
| **Azure Functions (isolated worker, .NET)** | Separate project in **Phase 2** when adding queue/Service Bus processing |
| **Class library** | Shared DTOs + Cosmos/Blob clients (optional cleanup) |

---

## 6. Azure subscription — provisioning checklist (conceptual)

Single **region**, single **resource group** (e.g. `rg-ai200-poc-{suffix}`), **budget alert** enabled.

1. Resource group  
2. **Azure Container Registry** (SKU aligned to budget; delete when idle)  
3. **Azure Container Apps** environment + app (locked POC host — §8)  
4. *(Optional learning only)* **Azure Container Instances** — one-off run of the same ACR image (`az container create`, public IP); **not** the main API (§8 D1)  
5. **Azure Cosmos DB for NoSQL** — capacity mode chosen for POC cost (serverless or minimal RU; validate portal)  
6. **Storage account** — Blob containers for uploads / artifacts  
7. **Azure OpenAI** and/or **Azure AI Foundry** project — embedding model deployment  
8. **Azure Key Vault** — keys / connection strings (API references via MI + RBAC where possible)  
9. **Log Analytics + Application Insights** for traces and **KQL**  
10. **Phase 2:** **Azure Service Bus** namespace + queue (or topic/subscription); **Azure Functions** + hosting plan / Flex consumption  
11. **Phase 2+ (optional):** **Azure Managed Redis**

**Resource group teardown (see §8):** When you **delete the resource group**, Azure deletes **all resources inside it** (Container Apps, Cosmos, storage, etc.) and **billing stops** for those resources (subject to any retention/soft-delete storage). Use this to **avoid runaway POC cost** when you pause work.

---

## 7. Microsoft Learn documentation index (exam study resources + containers)

Use these for implementation details and exam cross-training:

| Area | Documentation |
|------|----------------|
| Azure (hub) | [Azure documentation](https://learn.microsoft.com/en-us/azure/?product=featured) |
| Container Registry | [Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/) |
| Container Instances | [Container Instances](https://learn.microsoft.com/en-us/azure/container-instances/) |
| Container Apps | [Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/) (ACA + KEDA — strong AI-200 fit) |
| App Service | [App Service](https://learn.microsoft.com/en-us/azure/app-service/) |
| Azure Functions | [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/) |
| Azure Cosmos DB | [Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/) |
| Blob Storage | [Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/) |
| Microsoft Entra ID | [Microsoft Entra ID](https://learn.microsoft.com/en-us/azure/active-directory/) — **not in this POC** (§8); general AI-200 study |
| Key Vault | [Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/) |
| Azure Cache for Redis | [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/) |
| API Apps | [API Apps](https://learn.microsoft.com/en-us/azure/app-service/) (App Service–hosted APIs) |
| API Management | [API Management](https://learn.microsoft.com/en-us/azure/api-management/) — **not in this POC** (§8) |
| Event Hubs | [Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/) |
| Event Grid | [Event Grid](https://learn.microsoft.com/en-us/azure/event-grid/) |
| Service Bus | [Service Bus Messaging](https://learn.microsoft.com/en-us/azure/service-bus-messaging/) |
| Queue Storage | [Queue Storage](https://learn.microsoft.com/en-us/azure/storage/queues/) |

**Note:** The [AI-200 “Find documentation”](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200#study-resources) strip on Microsoft Learn does not always list **Container Apps**; this table includes **Container Apps** because the exam skills cover **ACA** and **KEDA**. Rows marked **not in this POC** reflect **§8** (API Management, Entra).

## 8. Locked design decisions (planning baseline)

These choices are **fixed** for this POC so implementation and companion docs stay consistent.

| ID | Decision | Status |
|----|----------|--------|
| **D1** | **Azure Container Apps** hosts the ASP.NET Core API (Docker image from **ACR**). | **Locked.** App Service / ACI are not alternate hosts for the main API in this POC (ACI remains optional as a **learning drill** only — see §2.9). |
| **D2** | **Azure Service Bus** is the messaging service for **Phase 2** async ingest (with **Azure Functions**). | **Locked.** Storage Queue is not part of this POC’s design (you can still read Learn docs for comparison). |
| **D3** | **Resource group (RG) teardown** for cost control | **Locked recommendation:** use **one** resource group (name pattern in companion [provisioning doc](POC-Azure-AI-200-phase1-azure-provisioning.md)). **Teardown** = **delete that resource group** in the portal or Azure CLI when you pause the POC for several days or finish a milestone, so **all resources inside it are deleted** and **most charges stop** (verify **Blob soft delete** / **Cosmos** backup retention so you are not surprised). **Rhythm:** either after each milestone or **weekly** if you iterate often — same mechanism. |
| **D4** | **API Management** | **Out of scope** for this POC (removed from architecture and stretches). |
| **D5** | **Microsoft Entra ID** (app registration / API protection) | **Out of scope** for this POC (removed from architecture and stretches). |

**What is an RG?** A **resource group** is a **container in Azure** that groups related resources (Cosmos, Storage, ACA, etc.) for **lifecycle and billing views**. It is **not** a compute runtime. **Deleting the RG** deletes (almost) everything inside it — that is **teardown**.

**Companion documents:** [Phase 1 — Azure provisioning (Spain)](POC-Azure-AI-200-phase1-azure-provisioning.md) · [Phase 1 — Visual Studio project](POC-Azure-AI-200-phase1-visual-studio.md)

---

## 9. Implementation phases (after design sign-off)

**Phase 1 — semantic search (sync)**  

1. Local **Web API** + **Dockerfile**; push to **ACR**; deploy to **Container Apps** (§8 D1); optional **ACI** drill with the same image.  
2. **Cosmos DB** — container(s) for chunk items + **vector indexing** per current docs.  
3. **Blob** container for **`.txt`/`.md`**; upload via API, Storage Explorer, or AzCopy.  
4. **Sync indexing:** endpoint that reads Blob → **chunk** → **embedding API** → **upsert** Cosmos.  
5. **Query** endpoint: embed user text → **vector search** in Cosmos → return **top K** + **scores** (optional confidence bands).  
6. **Key Vault** + **Managed Identity**.  
7. **Insights** + **KQL** queries.  

**Phase 2 — async ingest (locked: Service Bus + Functions)**  

8. **Azure Service Bus** + **Azure Functions** (trigger on message): enqueue after upload or from Event Grid; worker chunks → embed → Cosmos.  
9. Optional: **Event Grid** on blob created; optional **RAG** (completion after retrieval); optional **Redis** (Stretch C).  
10. **Teardown:** delete POC **resource group** when pausing; verify spend (§8 D3).

---

## 10. References (certification + this POC)

- [AI-200 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)  
- [Azure AI Foundry documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/) (supporting only)  
- [Training for Microsoft Foundry](https://learn.microsoft.com/en-us/training/azure/ai-foundry)  
- **This repo — design:** [POC-Azure-AI-200-task-and-design.md](POC-Azure-AI-200-task-and-design.md)  
- **This repo — Phase 1 Azure (Spain):** [POC-Azure-AI-200-phase1-azure-provisioning.md](POC-Azure-AI-200-phase1-azure-provisioning.md)  
- **This repo — Phase 1 Visual Studio:** [POC-Azure-AI-200-phase1-visual-studio.md](POC-Azure-AI-200-phase1-visual-studio.md)  

---
