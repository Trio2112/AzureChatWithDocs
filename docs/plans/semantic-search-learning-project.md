# AzureChatWithDocs — Semantic Search Learning Project

## Context

Brent is studying for the **AI-200** Azure exam and wants hands-on practice with the
two vector-search learning paths:

- Develop AI Solutions with Azure Cosmos DB
- Develop AI Solutions with Azure Database for PostgreSQL

Rather than have me generate a finished repo, the explicit goal is to **build this
together incrementally, session by session**, with me explaining the Azure/AI concepts
as we go and Brent actively writing/deciding parts himself — not watching me do it all.

Decisions already made together (via questions this session):
- **Vector store**: Cosmos DB NoSQL API first (vector indexing policy). Postgres/pgvector
  is a planned stretch goal later, swapping the storage layer to compare the two paths.
- **Stack**: C# / .NET throughout (ingestion, API, infra tooling) — aligns with AI-200's
  .NET SDK coverage.
- **Embeddings**: Azure OpenAI (`text-embedding-3-small`) — cheap at mock-corpus scale
  and matches what the exam actually covers (Azure OpenAI resource/deployment/quota).
- **Frontend**: simple vanilla HTML/JS chat UI, no framework — served as static files
  from the same ASP.NET Core project as the chat API, so it's one container to build/
  deploy (matches the Container Apps hosting path Brent linked).
- **Corpus theme**: fictional internal IT/HR knowledge base (policies, onboarding,
  troubleshooting articles) — good mix of overlapping-but-distinct topics for testing
  semantic vs. keyword retrieval later.
- **Environment**: Brent already has an Azure subscription and an empty GitHub repo
  ready; we'll connect to them as we go rather than provision now.

## Full roadmap (reference — executed across future sessions, not all at once)

Each phase = one teaching session. I explain the Azure concept, we decide the approach
together, and Brent does part of the hands-on work (not just review).

| Phase | Topic | Exam relevance |
|---|---|---|
| 0 | Repo scaffolding, solution structure, roadmap doc | — |
| 1 | Mock HTML corpus (IT/HR docs), authored together | Content prep for RAG |
| 2 | Chunking strategy (library in `Shared`) | Chunking/indexing strategies |
| 3 | Provision Azure OpenAI (Bicep), call embeddings API, inspect vectors | Azure OpenAI resource/deployment |
| 4 | Provision Cosmos DB NoSQL (Bicep, serverless, vector index policy), upsert chunks+vectors | Cosmos DB vector indexing |
| 5 | Vector search queries + RAG retrieval logic | Vector search / retrieval patterns |
| 6 | Chat API (ASP.NET Core Minimal API: retrieve + Azure OpenAI chat completion) | RAG orchestration |
| 7 | Frontend chat UI (vanilla HTML/JS) wired to the API | — |
| 8 | Dockerfile + Azure Container Apps hosting (Bicep) | Container Apps hosting path |
| 9 | Re-vectorization pipeline for added/changed content (content-hash diffing; likely an Azure Function w/ Blob trigger) | Event-driven ingestion pipelines |
| 10 | GitHub Actions CI/CD (build/test → build+push image → deploy to Container Apps) | — |
| 11+ (stretch) | Swap in PostgreSQL/pgvector to compare; hybrid search; Entra ID auth on the app; Application Insights; RAG quality eval; `azd down` cost-teardown habit | Postgres path + rounding out exam coverage |

## This session's scope: Phase 0 + Phase 1 only

Stop after this — do **not** start chunking/embeddings/Cosmos yet. Those are separate
future sessions per the roadmap above.

### Phase 0 — Scaffolding

1. `git init` in `C:\data\git\AzureChatWithDocs`; add a standard Visual Studio/.NET
   `.gitignore`.
2. Verify the .NET SDK is installed (`dotnet --version`); tell Brent what to install if
   missing rather than silently working around it.
3. Create the solution and three projects with `dotnet new`:
   - `src/AzureChatWithDocs.sln`
   - `src/Shared` (classlib) — chunking models/logic, shared between Ingestion and Api later
   - `src/Ingestion` (console) — will host the embed/upsert pipeline (Phase 3+)
   - `src/Api` (web, minimal API) — will host the chat endpoint + `wwwroot` static frontend (Phase 6+)
   - Wire project references (`Ingestion`→`Shared`, `Api`→`Shared`) now even though the
     code inside is still empty, so the solution builds cleanly from the start.
4. `README.md` at repo root: what this project is, the phase table above, how to run
   things (filled in incrementally as phases land), and links to the two Microsoft
   Learn paths Brent provided.
5. `docs/roadmap.md`: the same phase table, as our shared syllabus/checklist we tick off
   session by session.

### Phase 1 — Mock corpus

**Update (2026-08-16):** Brent doesn't want to spend time authoring sample documents —
it's not valuable study time. I write the full corpus; he reviews and steers.

1. Create `content/hr/` and `content/it/` folders.
2. I write realistic-length HTML docs (500-1000+ words, multiple `<h2>`/`<h3>`
   sections, lists, and at least a couple of tables) across both folders, with
   deliberate topic overlaps across HR/IT so later we can see semantic search
   disambiguate intent (e.g. remote work policy vs. VPN setup guide both mention VPN;
   code of conduct vs. acceptable use policy both mention confidentiality/data
   handling).
3. Brent reviews and asks for revisions/additions as needed.
4. No manifest/metadata file yet — that's introduced in Phase 2 when we design chunking
   and need per-doc IDs.

## Update log

- **2026-08-16:** Corpus authoring reassigned solely to Claude (see Phase 1 note
  above). Repo created as **public** at github.com/Trio2112/AzureChatWithDocs, pushed
  to `main`. Brent added new standing requirements: no secrets in code (Key Vault for
  anything sensitive), and every Azure resource this project uses must be creatable
  and destroyable via one standup script and one teardown script, portable to a
  different Azure subscription. This is substantial enough to be its own phase — the
  roadmap now has a new **Phase 3: Infra foundation** before any resource gets
  provisioned (Azure OpenAI/Cosmos DB/Container Apps phases shift down accordingly).
  **`docs/roadmap.md` is the live source of truth for phase numbering and status from
  here on** — this file's roadmap table above reflects the plan as originally
  approved, not the current state.

## Verification

- `dotnet build` on the new solution succeeds with three empty-but-wired-up projects.
- `git status` / `git log` show a clean initial commit (only after Brent confirms he
  wants it committed — I will not commit without asking, per standing instructions).
- Manual read-through of the seed HTML docs for consistency of structure.
