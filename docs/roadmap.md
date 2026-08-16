# Roadmap / syllabus

Our shared checklist. Each phase is one working session: we establish the underlying
Azure/AI concept, weigh the real tradeoffs, decide the approach together, and record
*why* — then the implementation follows. Check off phases as we complete them.

## How this roadmap is weighted, and why

An earlier draft was eight build phases with observability, evaluation and cost folded
into a single "stretch" row at the bottom. That ordering optimizes for getting a demo
working, which is the wrong target: assembling a RAG pipeline from documented
components is commodity work, and it is not what makes a system defensible.

This version promotes the parts that separate a system from a demo — evaluation,
access control, observability, cost, failure modes — into first-class phases,
positioned where they actually inform decisions rather than appended after the
decisions are frozen.

There is a second reason. In a codebase where most code is AI-authored, the
verification infrastructure *is* the senior engineering deliverable: tests, evals, CI
gates, traces and cost ceilings are the mechanism by which output you did not type by
hand becomes output you can trust and defend. Those phases are the point, not the
epilogue.

## Phases

| # | Phase | Exam relevance | Status |
|---|---|---|---|
| 0 | Repo scaffolding, solution structure, roadmap | — | ✅ Done |
| 1 | Mock HTML corpus (IT/HR docs) | Content prep for RAG | ✅ Done |
| 2 | Chunking strategy (`Shared` library) + supply-chain hardening | Chunking/indexing strategies | 🚧 In progress |
| 3 | **Golden question set** — 30–40 questions with expected source sections, authored against the corpus. No Azure spend | Evaluation groundwork | ⬜ |
| 4 | **Infra foundation:** portable standup/teardown (`azd` + Bicep), Key Vault, no secrets in code — **plus minimal CI** (build + test on PR) | Infra-as-code, secrets management | ⬜ |
| 5 | Azure OpenAI provisioning, embeddings API, inspect vectors | Azure OpenAI resource/deployment | ⬜ |
| 6 | Cosmos DB NoSQL (serverless, vector index policy), upsert chunks+vectors | Cosmos DB vector indexing | ⬜ |
| 7 | Vector search queries + RAG retrieval logic | Vector search / retrieval patterns | ⬜ |
| 8 | **RAG evaluation harness** — runs the Phase 3 questions, reports recall@k and MRR, A/Bs chunking profiles. Wired into CI as a gate | Production practice | ⬜ |
| 9 | Chat API (ASP.NET Core minimal API: retrieve + chat completion) | RAG orchestration | ⬜ |
| 10 | **Identity & document-level access control** — Entra ID auth, security trimming enforced *inside* the vector query | Enterprise RAG / security | ⬜ |
| 11 | **Observability** — App Insights, retrieval traces, token spend per request, latency breakdown, grounding failures | Monitoring AI solutions | ⬜ |
| 12 | **Cost model & scale analysis** — measured per-query and per-reindex cost, 10× and 100× projections, budget alerts | Capacity/cost planning | ⬜ |
| 13 | Frontend chat UI (vanilla HTML/JS) with citations back to source docs | — | ⬜ |
| 14 | Dockerfile + Azure Container Apps hosting | Container Apps hosting path | ⬜ |
| 15 | **Failure modes & resilience** — empty retrieval, 429 backoff, malformed docs, embedding model version drift, partial index state | Reliability practice | ⬜ |
| 16 | Re-vectorization pipeline for added/changed content (hash diffing; Function w/ Blob trigger) | Event-driven ingestion | ⬜ |
| 17 | Full CI/CD (build/test/eval gates → image → Container Apps) | — | ⬜ |
| 18+ | Stretch: PostgreSQL/pgvector comparison; hybrid search; reranking; answer-groundedness eval | Postgres path | ⬜ |

## Document-level access control (Phase 10, but decided earlier)

This is the single most common reason enterprise RAG pilots stall at security review,
and it cannot be bolted on afterwards.

The naive approach — retrieve top-k, then drop results the user isn't allowed to see —
is wrong twice over. It leaks information through the gaps (a user learns a restricted
document exists and roughly what it's about), and it silently degrades answers, because
the k slots were spent on documents that got discarded. **Authorization must be a
predicate inside the vector query, not a filter after it.**

That constrains decisions in earlier phases, so they're made now rather than revisited:

- **Phase 2** — every chunk carries `SecurityLabels`, sourced from a required
  `<meta name="access">` tag on the source document. **Fail closed:** a missing or
  empty tag grants access to nobody, and there is no implicit `public` — universal
  readability is an explicit label a human chose to write. Because silent
  fail-closed just makes documents mysteriously invisible, ingest *rejects* an
  untagged document, logs it, and counts it in the CLI summary, so the failure is
  loud. The corpus gains a few deliberately restricted documents (compensation bands,
  an investigation record) so the unrestricted path is never the only one tested.
- **Phase 6** — Cosmos partition key and vector index policy must support a filtered
  vector search. Filter selectivity interacts with ANN recall, and that tradeoff gets
  measured, not assumed.
- **Phase 7** — retrieval takes the caller's effective labels as a required argument.
  There is no overload that omits it.
- **Phase 8** — the golden question set includes questions whose answers live in
  restricted documents. The eval asserts an unauthorized caller gets *no* leak, not
  merely a lower score.
- **Phase 10** — Entra ID supplies real group membership, and an end-to-end test proves
  a restricted document never reaches an unauthorized caller.

## Anti-drift mechanisms

Phases 8, 11, 12 and 15 are the ones that make this a system rather than a demo. They
are also the ones most likely to be quietly deferred, because by the time they come due
there is a working chatbot and the interesting part feels finished. These are the
structural defenses, not good intentions:

**1. The golden question set is authored first (Phase 3), before any Azure resource
exists.** The hardest-to-motivate artifact is the one with no dependencies — 30–40
questions against 10 documents can be written today for zero cost. Phase 8 then becomes
"wire it up and measure," which is an afternoon, rather than "invent an eval from
scratch," which is a weekend that never comes.

**2. CI exists from Phase 4, long before there is much to gate.** A build-and-test
workflow is trivial to add then and nearly impossible to retrofit enthusiasm for later.
The moment the eval harness exists it becomes a CI gate, which means a retrieval
regression breaks the build rather than going unnoticed.

**3. Later phases have earlier phases as hard exit criteria.** These are not
suggestions; a phase is not done until they hold:

| Phase | Cannot be marked done until |
|---|---|
| 9 — Chat API | the Phase 8 eval harness runs against it end-to-end and reports a baseline |
| 13 — Frontend | every answer renders citations resolving to a real source section |
| 14 — Container Apps | Phase 11 traces are visible from the deployed instance, not just locally |
| 16 — Re-vectorization | Phase 15's malformed-document and partial-index cases are covered |
| 17 — CI/CD | the eval gate fails the build on a recall@k regression beyond an agreed threshold |

**4. Instrument continuously; Phase 11 is the coherence pass, not the first pass.**
Every phase that makes a network call adds its trace and token-count at the time it is
written. Phase 11 makes it coherent and dashboards it. Retrofitting instrumentation
across a finished codebase is the reason it usually doesn't happen.

**5. Cost is tracked from the first Azure resource.** Phase 4 sets an Azure budget with
alerts. Phase 12 turns running actuals into a model and projections — it is analysis of
data already being collected, not a from-scratch investigation.

**6. The README's claims are the tripwire.** It states this project has measured
retrieval quality, a cost model, and documented failure behavior. Any of those still
unbuilt must be visibly marked pending in the README. An unmarked false claim in a
public repo is a real cost, which is the point.

## Running practice: decision records

Every phase produces a plan document in [`docs/plans/`](plans/) recording the decision,
the alternatives considered, the evidence, and the residual risk knowingly accepted.
These are as much a deliverable as the code.

The template is the AngleSharp supply-chain review in
[`docs/plans/phase-2-chunking.md`](plans/phase-2-chunking.md): what was chosen, what was
rejected and why, what the measured data said, what risk remains and why it was
acceptable. Code shows what was built; these show why, and they are the artifact that
survives when the code is regenerated.

## Key decisions (see plan docs for full rationale)

- Vector store: Cosmos DB NoSQL API first; Postgres/pgvector later as a comparison.
- Stack: C# / .NET 10 throughout.
- Embeddings: Azure OpenAI `text-embedding-3-small` (cl100k_base, 1536 dims).
- Chunking: structure-aware, 350 target / 500 max real tokens, heading-path prefixed.
  Chunk records carry a chunking-profile ID so competing strategies can be indexed side
  by side and A/B'd in Phase 8 without a destructive reindex.
- Authorization is a predicate inside the vector query, never a post-retrieval filter.
- Frontend: vanilla HTML/JS served as static files from the `Api` project — one
  container, deployed to Azure Container Apps.
- Corpus: fictional internal IT/HR knowledge base with deliberately overlapping topics
  and mixed sensitivity, so retrieval has to discriminate rather than pattern-match.
- Repo is public (github.com/Trio2112/AzureChatWithDocs) — no secrets ever committed;
  Azure secrets live in Key Vault, referenced by config, not hardcoded.
- All Azure resources must be creatable and destroyable via a single standup/teardown
  script pair, portable to a different Azure subscription.
- Dependencies are reviewed before adoption (advisories, provenance, maintenance
  health, residual risk) and pinned with a lockfile plus `NuGetAudit`.
