# AzureChatWithDocs

A retrieval-augmented generation (RAG) chatbot over a corpus of internal HR/IT
documents — built on Azure Cosmos DB vector search, Azure OpenAI, and Azure Container
Apps, in C# / .NET 10.

It is being built to serve two purposes at once, and the second one shapes almost every
decision in it.

## 1. Hands-on preparation for AI-200

Working coverage of *Develop AI Solutions in Microsoft Azure* — chunking and indexing
strategy, embeddings, Cosmos DB vector indexing, retrieval patterns, RAG orchestration,
Container Apps hosting, and event-driven reingestion — following these Microsoft Learn
paths:

- [Develop AI Solutions with Azure Cosmos DB](https://learn.microsoft.com/en-us/training/paths/develop-ai-solutions-azure-cosmos-db/)
- [Develop AI Solutions with Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/training/paths/develop-ai-solutions-azure-database-postgresql/)
- [Implement Container App Hosting on Azure](https://learn.microsoft.com/en-us/training/paths/implement-container-app-hosting-azure/)

## 2. A defensible demonstration of AI-directed system design

Assembling a RAG pipeline is commodity work now. Chunk, embed, cosine-similarity, stuff
a prompt — the components are well documented and an agent can write them quickly. That
is not the interesting part, and this project does not pretend it is.

The interesting part is everything that determines whether the result can be operated,
trusted, changed, and paid for. So this repo is deliberately weighted toward the work
that a demo skips:

**Decisions are recorded, not just made.** Every phase produces a plan document in
[`docs/plans/`](docs/plans/) capturing the decision, the alternatives weighed, the
evidence behind the choice, and the residual risk knowingly accepted. Code shows what
was built; these show why. See the AngleSharp supply-chain review in
[`phase-2-chunking.md`](docs/plans/phase-2-chunking.md) as the template — advisory
history, distribution integrity, maintainer concentration, CI posture, and an explicit
statement of what risk was accepted and what mitigates it.

**Decisions are grounded in measurement, not vibes.** Chunk sizing wasn't picked from a
blog post: the corpus was measured first (every `<h2>` section is 46–158 tokens, so the
real problem is *merging* undersized sections, not splitting oversized ones), and chunk
budgets are counted in real `cl100k_base` tokens rather than a `chars/4` approximation.
Phase 8 then re-tests that choice against a golden question set instead of trusting it.

**Retrieval quality is measured.** A RAG system whose retrieval quality nobody has
quantified is a demo. The evaluation harness is a first-class phase positioned right
after retrieval works, so chunking and top-k are tuned against recall@k and MRR. Chunk
records carry a chunking-profile ID specifically so competing strategies can be indexed
side by side and compared without a destructive reindex.

**Operability is designed in, not appended.** Observability lands with the API phase,
not in a stretch goal: retrieval traces, token spend per request, latency breakdown,
grounding failures. Cost gets its own phase with measured per-query and per-reindex
numbers and projections at 10× and 100× corpus size. Failure modes get a systematic
pass — empty retrieval, 429 backoff, malformed documents, embedding model version
drift, partial index state.

**The supply chain is treated as part of the system.** Dependencies are reviewed before
adoption and pinned with `packages.lock.json`, `NuGetAudit`, and NuGet package source
mapping. Every Azure resource is creatable and destroyable by one script pair, portable
to a different subscription. No secrets in code, ever — Key Vault, referenced by config.

**On how it is built:** most of the code here is AI-authored, under direction, by
design. In that model the verification infrastructure *is* the engineering deliverable —
tests, evals, CI gates, traces and cost ceilings are what make code you didn't type by
hand into code you can defend. The roadmap is weighted accordingly.

## Architecture

```
content/*.html
     │
     ▼
┌──────────────┐   chunks    ┌─────────────────┐  vectors  ┌──────────────────┐
│  Ingestion   │────────────▶│  Azure OpenAI   │──────────▶│    Cosmos DB     │
│  (chunking)  │             │   embeddings    │           │  NoSQL + vector  │
└──────────────┘             └─────────────────┘           │   index policy   │
                                                           └──────────────────┘
                                                                     │
                                    question                         │ top-k
                                        │                            ▼
┌──────────────┐             ┌─────────────────┐           ┌──────────────────┐
│  Chat UI     │────────────▶│   Api (minimal) │◀──────────│    retrieval     │
│  (wwwroot)   │◀────────────│  RAG orchestr.  │           └──────────────────┘
└──────────────┘   answer    └─────────────────┘
                                     │ chat completion
                                     ▼
                             ┌─────────────────┐
                             │  Azure OpenAI   │
                             └─────────────────┘

hosted as one container on Azure Container Apps
```

## Repo layout

```
content/        Mock HTML corpus (the "documents" we chunk and index)
src/            .NET 10 solution
  Shared/       Chunking models/logic shared by Ingestion and Api
  Shared.Tests/ xUnit tests + fixtures for the chunking library
  Ingestion/    Console app: chunk content, generate embeddings, upsert to Cosmos DB
  Api/          ASP.NET Core minimal API: chat endpoint + static chat UI (wwwroot)
infra/          Bicep infrastructure-as-code (Phase 4)
eval/           Golden question set (Phase 3) + retrieval metrics harness (Phase 8)
.github/        GitHub Actions CI (Phase 4) and CD (Phase 17)
docs/
  roadmap.md    Phase-by-phase plan and current status
  plans/        Per-phase decision records
```

## How this project is built

One phase at a time, plan-first — not generated in one shot. Each phase begins by
establishing the concept and the real tradeoffs, decides an approach explicitly, and
records the reasoning before code is written. See
[`docs/roadmap.md`](docs/roadmap.md) for the phase list and status,
[`CONTRIBUTING.md`](CONTRIBUTING.md) for the working conventions, and
[`docs/plans/`](docs/plans/) for the decision records.

## Running it

Filled in as each phase lands. Currently:

```bash
dotnet restore src/AzureChatWithDocs.slnx --locked-mode
dotnet build   src/AzureChatWithDocs.slnx
dotnet test    src/AzureChatWithDocs.slnx
```

## Cost posture

A dev-only learning project. Wherever Azure offers a choice we take the
cheapest/serverless/burstable tier, and resources are torn down when not in active use
(`azd down`). Phase 12 replaces this paragraph with measured numbers — per-query cost,
per-reindex cost, and what both look like at 10× and 100× the corpus.
