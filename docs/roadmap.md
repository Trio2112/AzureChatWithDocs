# Roadmap / syllabus

Our shared checklist. Each phase is one teaching session: I explain the underlying
Azure/AI concept, we decide the approach together, and you do part of the hands-on
work — not just review finished code. Check off phases as we complete them.

| # | Phase | Exam relevance | Status |
|---|---|---|---|
| 0 | Repo scaffolding, solution structure, roadmap doc | — | ✅ Done |
| 1 | Mock HTML corpus (IT/HR docs) | Content prep for RAG | ✅ Done |
| 2 | Chunking strategy (library in `Shared`) | Chunking/indexing strategies | ⬜ Not started |
| 3 | Provision Azure OpenAI (Bicep), call embeddings API, inspect vectors | Azure OpenAI resource/deployment | ⬜ Not started |
| 4 | Provision Cosmos DB NoSQL (Bicep, serverless, vector index policy), upsert chunks+vectors | Cosmos DB vector indexing | ⬜ Not started |
| 5 | Vector search queries + RAG retrieval logic | Vector search / retrieval patterns | ⬜ Not started |
| 6 | Chat API (ASP.NET Core minimal API: retrieve + Azure OpenAI chat completion) | RAG orchestration | ⬜ Not started |
| 7 | Frontend chat UI (vanilla HTML/JS) wired to the API | — | ⬜ Not started |
| 8 | Dockerfile + Azure Container Apps hosting (Bicep) | Container Apps hosting path | ⬜ Not started |
| 9 | Re-vectorization pipeline for added/changed content (content-hash diffing; likely an Azure Function with a Blob trigger) | Event-driven ingestion pipelines | ⬜ Not started |
| 10 | GitHub Actions CI/CD (build/test → build+push image → deploy to Container Apps) | — | ⬜ Not started |
| 11+ | Stretch: swap in PostgreSQL/pgvector to compare; hybrid search; Entra ID auth; Application Insights; RAG quality eval; `azd down` cost-teardown habit | Postgres path + rounding out exam coverage | ⬜ Not started |

## Key decisions (see plan doc for full rationale)

- Vector store: Cosmos DB NoSQL API first; Postgres/pgvector later as a comparison.
- Stack: C# / .NET throughout.
- Embeddings: Azure OpenAI `text-embedding-3-small`.
- Frontend: vanilla HTML/JS served as static files from the `Api` project — one
  container, deployed to Azure Container Apps.
- Corpus: fictional internal IT/HR knowledge base with deliberately overlapping topics.
