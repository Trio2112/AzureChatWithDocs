# Roadmap / syllabus

Our shared checklist. Each phase is one working session: we establish the underlying
Azure/AI concept, weigh the real tradeoffs, decide the approach together, and record
*why* — then the implementation follows. Check off phases as we complete them.

## How this roadmap is weighted, and why

An earlier draft of this roadmap was eight build phases with observability, evaluation
and cost folded into a single "stretch" row at the bottom. That ordering optimizes for
getting a demo working, which is the wrong target: assembling a RAG pipeline from
documented components is commodity work, and it is not what makes a system defensible.

This version promotes the parts that separate a system from a demo — evaluation,
observability, cost, failure modes — into first-class phases, positioned where they
actually inform decisions rather than appended after the decisions are frozen.

There is a second reason. In a codebase where most code is AI-authored, the
verification infrastructure *is* the senior engineering deliverable: tests, evals, CI
gates, traces and cost ceilings are the mechanism by which output you did not type by
hand becomes output you can trust and defend. Those phases are the point, not the
epilogue.

## Phases

| # | Phase | Why it matters | Exam relevance | Status |
|---|---|---|---|---|
| 0 | Repo scaffolding, solution structure, roadmap | — | — | ✅ Done |
| 1 | Mock HTML corpus (IT/HR docs) | Realistic input with deliberately overlapping topics | Content prep for RAG | ✅ Done |
| 2 | Chunking strategy (`Shared` library) + supply-chain hardening | The single biggest lever on retrieval quality; first third-party deps enter the repo | Chunking/indexing strategies | 🚧 In progress |
| 3 | **Infra foundation:** portable standup/teardown for all Azure resources (`azd` + Bicep), Key Vault, no secrets in code | Proves the system can be stood up and destroyed by someone who isn't us | Infra-as-code, secrets management | ⬜ |
| 4 | Azure OpenAI provisioning, embeddings API, inspect vectors | — | Azure OpenAI resource/deployment | ⬜ |
| 5 | Cosmos DB NoSQL (serverless, vector index policy), upsert chunks+vectors | — | Cosmos DB vector indexing | ⬜ |
| 6 | Vector search queries + RAG retrieval logic | — | Vector search / retrieval patterns | ⬜ |
| 7 | **RAG evaluation harness** — golden question set, retrieval metrics (recall@k, MRR), tune chunking against measured results | Without this, every retrieval-quality decision is guesswork. This is what converts "I picked 350 tokens" into "I measured 350 against alternatives." | Beyond exam scope — production practice | ⬜ |
| 8 | Chat API (ASP.NET Core minimal API: retrieve + chat completion) | — | RAG orchestration | ⬜ |
| 9 | **Observability** — Application Insights, retrieval traces, token spend per request, latency breakdown, grounding failures | Attached to the API phase deliberately, not deferred. You cannot operate what you cannot see. | Monitoring AI solutions | ⬜ |
| 10 | **Cost model & scale analysis** — measured per-query and per-reindex cost, projections at 10× and 100× corpus size, cost ceilings | Turns the README's "cost-conscious" claim into numbers that can be defended in a review | Capacity/cost planning | ⬜ |
| 11 | Frontend chat UI (vanilla HTML/JS) with citations back to source docs | — | — | ⬜ |
| 12 | Dockerfile + Azure Container Apps hosting | — | Container Apps hosting path | ⬜ |
| 13 | **Failure modes & resilience** — no relevant results, rate limiting/429 backoff, malformed documents, embedding model version drift, partial index state | Systematic pass over what breaks in production. Model-version drift in particular is a nasty surprise most teams hit by accident. | Reliability practice | ⬜ |
| 14 | Re-vectorization pipeline for added/changed content (content-hash diffing; Azure Function w/ Blob trigger) | Incremental reindexing is a genuine production problem that most portfolio projects skip entirely | Event-driven ingestion pipelines | ⬜ |
| 15 | GitHub Actions CI/CD (build/test/eval gates → image → Container Apps) | CI runs the eval harness as a gate, not just unit tests | — | ⬜ |
| 16+ | Stretch: PostgreSQL/pgvector comparison; hybrid search; Entra ID auth; reranking | Postgres path + rounding out exam coverage | ⬜ |

## Running practice: decision records

Every phase produces a plan document in [`docs/plans/`](plans/) that records the
decision, the alternatives considered, the evidence, and the residual risk knowingly
accepted. These are as much a deliverable as the code.

The template is the AngleSharp supply-chain review in
[`docs/plans/phase-2-chunking.md`](plans/phase-2-chunking.md): what was chosen, what
was rejected and why, what the measured data said, what risk remains and why it was
acceptable. Code shows what was built; these show why, and they are the artifact that
survives when the code is regenerated.

## Key decisions (see plan docs for full rationale)

- Vector store: Cosmos DB NoSQL API first; Postgres/pgvector later as a comparison.
- Stack: C# / .NET 10 throughout.
- Embeddings: Azure OpenAI `text-embedding-3-small` (cl100k_base, 1536 dims).
- Chunking: structure-aware, 350 target / 500 max real tokens, heading-path prefixed.
  Chunk records carry a chunking-profile ID so competing strategies can be indexed
  side by side and A/B'd in Phase 7 without a destructive reindex.
- Frontend: vanilla HTML/JS served as static files from the `Api` project — one
  container, deployed to Azure Container Apps.
- Corpus: fictional internal IT/HR knowledge base with deliberately overlapping topics,
  so retrieval has to discriminate rather than pattern-match.
- Repo is public (github.com/Trio2112/AzureChatWithDocs) — no secrets ever committed;
  Azure secrets live in Key Vault, referenced by config, not hardcoded.
- All Azure resources must be creatable and destroyable via a single standup/teardown
  script pair, portable to a different Azure subscription.
- Dependencies are reviewed before adoption (advisories, provenance, maintenance
  health, residual risk) and pinned with a lockfile plus `NuGetAudit`.
