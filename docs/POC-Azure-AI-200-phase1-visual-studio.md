# Phase 1 — Create the solution in Visual Studio

**Goal:** A **Dockerized ASP.NET Core Web API** you can push to **ACR** and run on **Azure Container Apps**, aligned with [POC-Azure-AI-200-task-and-design.md](POC-Azure-AI-200-task-and-design.md).  
**Stack:** **.NET 10** (this repo’s choice for practice). **Language:** C#. Comments and identifiers in English.

### .NET 10 — is it viable for this POC?

**Yes.** The POC runs your app **inside a container** you build; Azure Container Apps runs **whatever image** you publish as long as it listens on the configured port. **.NET 10** is appropriate for **learning and practice**. Use the **official ASP.NET runtime base images** (`mcr.microsoft.com/dotnet/aspnet:10.0`, etc.) in the Dockerfile — Visual Studio generates compatible Dockerfiles when you target .NET 10.

**Practical notes:** Install the **[.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)** and a **Visual Studio** build that includes the **.NET 10** target framework in the project wizard (update VS if **.NET 10** does not appear in the framework list). If a specific Azure library lags on .NET 10 for a week, check NuGet for a compatible prerelease or temporarily pin that package — uncommon for mainstream Azure SDKs once GA.

---

## 1. Prerequisites (install once)

1. **Visual Studio 2022** (latest **17.x** update that supports **.NET 10** in templates) — workload **ASP.NET and web development**.  
2. Same installer → optional but recommended: **Azure development** (publishing, dependencies).  
3. **Docker Desktop for Windows** (WSL2 backend recommended) — required to **build and run** the container image locally.  
4. **.NET 10 SDK** — from [Download .NET 10](https://dotnet.microsoft.com/download/dotnet/10.0) (ensure `dotnet --list-sdks` shows a **10.x** SDK).  
5. Sign in to Visual Studio with the **same Microsoft account** as your **Azure subscription** (optional but speeds publish and Azure-connected services).

---

## 2. Create the Web API project

1. **File** → **New** → **Project**.  
2. Search **ASP.NET Core Web API** → select the C# template → **Next**.  
3. **Project name:** e.g. `Ai200SemanticSearch.Api` (PascalCase, no spaces).  
4. **Location:** your repo folder (e.g. under `Prueba Concepto AI-200\src`).  
5. **Solution name:** e.g. `Ai200SemanticSearch` — **Create**.  
6. **Framework:** **.NET 10**.  
7. **Authentication:** **None** for Phase 1 (Entra is **out of scope** per design **§8**).  
8. **Configure for HTTPS:** checked.  
9. **Enable Docker:** checked (**OS:** **Linux**).  
10. **Enable OpenAPI support:** checked (Swagger).  
11. **Do not** use “Enlist in .NET Aspire” for this minimal POC unless you already want that stack.  
12. **Create**.

Visual Studio generates the API project and a **Dockerfile** (Linux).

### Docker / container options in the wizard — what to choose?

- **Enable Docker** (Linux) → **Yes.** That is the right choice for this POC. It adds a **Dockerfile** at the project root (multi-stage build: SDK to compile, runtime image to run). This matches the design path: **Dockerfile → ACR → Container Apps**.

- If the UI offers **Docker** vs **Docker Compose** (or “single container” vs “compose”): choose **Docker** / **Dockerfile** / **one project, one container** for Phase 1. **Docker Compose** is only useful if you intentionally run **multiple** containers together locally (e.g. API + local database); you do **not** need it for the initial POC.

- You do **not** need a separate “cloud-only” container build type for this exercise. The **exam-aligned** flow is: **you own the Dockerfile**, build locally or in CI, push to **ACR**.

- After creation, confirm `Dockerfile` uses **.NET 10** base images (e.g. `mcr.microsoft.com/dotnet/aspnet:10.0` and matching `sdk:10.0` in the build stage). If VS generated an older tag, change the **10.0** image tags to match your SDK.

---

## 3. Verify Docker and first run

1. Set **Solution Configuration** to **Debug**.  
2. In the debug toolbar, change launch profile from **https** to **Docker** (if multiple profiles exist).  
3. **F5** — VS builds the image, starts the container, opens the browser. Confirm Swagger or a test endpoint responds.  
4. If Docker errors appear: ensure **Docker Desktop** is running; in Docker settings, **Use the WSL 2 based engine** if applicable.

---

## 4. Project layout (recommended early)

Keep Phase 1 simple; you can refactor later.

| Area | Suggestion |
|------|------------|
| **Endpoints** | Minimal APIs in `Program.cs` **or** Controllers folder — pick one style and stay consistent. |
| **Configuration** | `appsettings.json` for non-secret defaults; **User Secrets** for local dev secrets (right-click project → **Manage User Secrets**). |
| **Azure SDKs** | Add NuGet packages when you implement: Cosmos, Blob, Key Vault, OpenAI client, Application Insights. |
| **Dockerfile** | Keep the VS-generated file unless you need multi-stage tweaks for smaller images; ensure **EXPOSE** and **ASPNETCORE_URLS** match **Container Apps** ingress port. |

---

## 5. Connect to Azure resources (implementation order)

This mirrors [POC-Azure-AI-200-task-and-design.md](POC-Azure-AI-200-task-and-design.md) **§9** Phase 1:

1. **Cosmos DB** — SDK + vector query + indexing policy aligned to embedding dimensions.  
2. **Blob** — `BlobServiceClient`, container `corpus`.  
3. **Indexing endpoint** — read blob → chunk → call embedding → upsert Cosmos.  
4. **Search endpoint** — embed query → vector search → return top K + scores.  
5. **Key Vault** — `DefaultAzureCredential` / managed identity; avoid secrets in `appsettings.json` committed to git.  
6. **Application Insights** — add SDK or OpenTelemetry exporter; verify traces in portal.

---

## 6. Publish image to ACR (from Visual Studio)

1. Right-click project → **Publish** → **Azure** → **Azure Container Registry** → select your registry (create in Azure first per [Phase 1 provisioning](POC-Azure-AI-200-phase1-azure-provisioning.md)).  
2. Finish the wizard; VS builds and **pushes** the image.  
3. In **Azure Portal** → **Container Apps** → your app → **Revision management** → point the container to the **new image tag** if not auto-updated.

CLI alternative: `docker tag` / `docker push` after `az acr login` — use whatever you prefer for exam practice.

---

## 7. Optional: add a class library

**Add** → **New Project** → **Class Library** (e.g. `Ai200SemanticSearch.Core`) — **Framework: .NET 10** — for DTOs and shared clients; add a **project reference** from the API. Not mandatory on day one.

---

## 8. Git and secrets

Add to **`.gitignore`:** `**/appsettings.Development.json` only if it contains secrets (prefer **User Secrets** instead). Never commit **Key Vault** keys or **OpenAI** keys.
