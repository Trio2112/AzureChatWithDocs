# Roadmap / syllabus

Our shared checklist. Each phase is one teaching session: I explain the underlying
Azure/AI concept, we decide the approach together, and you do part of the hands-on
work — not just review finished code. Check off phases as we complete them.

| # | Phase | Exam relevance | Status |
|---|---|---|---|
| 0 | Repo scaffolding, solution structure, roadmap doc | — | ✅ Done |
| 1 | Mock HTML corpus (IT/HR docs) | Content prep for RAG | ✅ Done |
| 2 | Chunking strategy (library in `Shared`) | Chunking/indexing strategies | ⬜ Not started |
| 3 | **Infra foundation:** portable standup/teardown for all Azure resources (likely `azd` + Bicep), Key Vault for secrets, no secrets in code | Infra-as-code, secrets management | ⬜ Not started |
| 4 | Provision Azure OpenAI (via the Phase 3 foundation), call embeddings API, inspect vectors | Azure OpenAI resource/deployment | ⬜ Not started |
| 5 | Provision Cosmos DB NoSQL (serverless, vector index policy), upsert chunks+vectors | Cosmos DB vector indexing | ⬜ Not started |
| 6 | Vector search queries + RAG retrieval logic | Vector search / retrieval patterns | ⬜ Not started |
| 7 | Chat API (ASP.NET Core minimal API: retrieve + Azure OpenAI chat completion) | RAG orchestration | ⬜ Not started |
| 8 | Frontend chat UI (vanilla HTML/JS) wired to the API | — | ⬜ Not started |
| 9 | Dockerfile + Azure Container Apps hosting (added to the Phase 3 foundation) | Container Apps hosting path | ⬜ Not started |
| 10 | Re-vectorization pipeline for added/changed content (content-hash diffing; likely an Azure Function with a Blob trigger) | Event-driven ingestion pipelines | ⬜ Not started |
| 11 | GitHub Actions CI/CD (build/test → build+push image → deploy to Container Apps, using Phase 3's deployment credentials) | — | ⬜ Not started |
| 12+ | Stretch: swap in PostgreSQL/pgvector to compare; hybrid search; Entra ID auth; Application Insights; RAG quality eval | Postgres path + rounding out exam coverage | ⬜ Not started |

## Key decisions (see plan doc for full rationale)

- Vector store: Cosmos DB NoSQL API first; Postgres/pgvector later as a comparison.
- Stack: C# / .NET throughout.
- Embeddings: Azure OpenAI `text-embedding-3-small`.
- Frontend: vanilla HTML/JS served as static files from the `Api` project — one
  container, deployed to Azure Container Apps.
- Corpus: fictional internal IT/HR knowledge base with deliberately overlapping topics.
- Repo is public (github.com/Trio2112/AzureChatWithDocs) — no secrets ever committed;
  Azure secrets live in Key Vault, referenced by config, not hardcoded.
- All Azure resources for this project must be creatable and destroyable via a single
  standup/teardown script pair, portable to a different Azure subscription.
