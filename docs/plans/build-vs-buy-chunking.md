# Decision record — build vs. buy: chunking and ingestion

*Recorded 2026-08-17. This is a decision record, not a phase plan. It exists because
"we wrote our own chunker" is a claim that invites the question "why?", and the answer
should be on the record rather than reconstructed later.*

## The question

Phase 2 builds a custom structure-aware chunking library. Managed alternatives exist and
are mature. Do organizations actually write their own chunkers, and should we?

Short answer: **most don't, and for a production deployment we probably shouldn't
either.** We are building one anyway, for reasons that are specific and defensible — and
the honest version of that argument requires knowing precisely what we're declining.

## What the managed options actually do

### Azure AI Search — integrated vectorization

The Azure-native answer. An indexer pulls from a supported data source, a skillset
chunks and embeds, and the result lands in a search index — chunk → embed → index with
no pipeline code. Verified against Microsoft Learn, 2026-08-17:

**Text Split skill** (`Microsoft.Skills.Text.SplitSkill`) — explicitly documented as
**non-billable, with no Foundry Tools key requirement**, available in all regions and
tiers. Parameters:

| Parameter | Values |
|---|---|
| `textSplitMode` | `pages` or `sentences` |
| `maximumPageLength` | characters (min 300, max 50000, default 5000) or token limit |
| `pageOverlapLength` | leading chars/tokens carried from the previous page |
| `unit` | `characters` (default) or `azureOpenAITokens` |
| `azureOpenAITokenizerParameters.encoderModelName` | `cl100k_base` (default), `r50k_base`, `p50k_base`, `p50k_edit` |

Outputs `textItems`, `offsets`, `lengths`, and `ordinalPositions`. Microsoft's documented
recommendation for embedding models is a 512-token page length. Notably, the skill
implements tiktoken via **SharpToken and `Microsoft.ML.Tokenizers`** — the same library
Phase 2 selected, which is mild validation of that choice. It does **not** support
`o200k_base`.

Structure-aware chunking in Azure AI Search comes from a different path — the Azure
Content Understanding skill or document parsing modes — not from Text Split.

### What else the platform absorbs

This is the part worth being uncomfortable about. Compare against our own roadmap:

| Our phase | Managed equivalent |
|---|---|
| **2** — Chunking library | Text Split skill: configuration, free |
| **11** — Document-level access control | Foundry IQ, documented as "permission-aware knowledge bases" |
| **16** — 429 backoff and retry | "Batching and retry logic is built in (non-configurable)" |
| **17** — Re-vectorization pipeline | Indexer on a schedule picks up changed documents automatically |

Four phases. The last one stings most: Phase 17's content-hash diffing — described
elsewhere in this repo as one of the project's strongest differentiators — is a scheduled
indexer in the managed path. Index projections also provide the parent/child chunk
pattern we would otherwise hand-build.

### Non-Azure options

- **Unstructured.io** — closest thing to a dedicated chunking service; partition +
  chunk-by-title across PDF, HTML, Office formats.
- **Docling** (IBM, OSS), **LlamaParse**, **Reducto**, **Chunkr** — document parsing and
  chunking services.
- **LangChain / LlamaIndex** — `RecursiveCharacterTextSplitter`,
  `HTMLHeaderTextSplitter` (approximately what we designed), `SemanticChunker`.
- **AWS Bedrock Knowledge Bases**, **Vertex AI RAG Engine** — managed end-to-end RAG
  with fixed/semantic/hierarchical chunking as a configuration choice.

## Why we are building one anyway

Three reasons, in descending order of how much they'd survive scrutiny.

### 1. The vector store choice already foreclosed the managed path

**Azure Cosmos DB NoSQL has no integrated chunking or vectorization pipeline.** Its
vector search operates on embeddings the application has already generated and written.
Choose Cosmos as the vector store and you own ingestion — chunking, embedding calls,
retry, change detection — by construction.

This is the actual rule, and it generalizes: **the vector store determines whether you
write a chunker.** Azure AI Search hands you an ingestion pipeline. Cosmos DB hands you a
place to put vectors.

We chose Cosmos because AI-200's syllabus is built on it. That is a sound reason for a
learning project and *not* the reason anyone should choose it for production RAG.

### 2. Text Split has no structure-aware mode, and structure is our whole design

Both `textSplitMode` values — `pages` and `sentences` — are **size-based**. Neither
understands document structure. Phase 2's design depends on things Text Split does not
do:

- **Heading-scoped sections** at arbitrary depth (`h1`–`h6`), with the heading path
  driving chunk boundaries.
- **Heading-path prefixing** on chunk text before embedding — the cheap approximation of
  contextual retrieval.
- **Table integrity**, keeping the PTO accrual table whole with its header row, and
  repeating the header row when an oversized table must be split by row.
- **Merging undersized sections** toward a target. Our corpus sections are 46–158
  tokens; a size-based splitter with a 512-token page would emit one blurry chunk per
  document and defeat the point.
- **`SecurityLabels` and a chunking `ProfileId`** on each chunk record, for fail-closed
  authorization and side-by-side A/B of chunking strategies.

Getting structure-aware behavior inside Azure AI Search means the Content Understanding
or document-parsing path, which is a different and larger commitment than a config knob.

### 3. You cannot credibly evaluate a managed option you've never implemented

"Use the Text Split skill" is a received opinion until you have measured what
`maximumPageLength` and `pageOverlapLength` do to recall on your own corpus. Phase 8's
eval harness makes that measurable, and Phase 2 is what gives it something to compare
against.

## Honest limitations of our approach

Two, and the first is significant.

**Our corpus is unrealistically clean, and that is why a custom chunker is cheap here.**
The `content/` documents are hand-authored, well-nested HTML with `<thead>` on every
table. Real internal HR and IT documentation is PDFs, Word files, scanned policy
documents, and slide decks. The hard problem in production ingestion is not chunking —
it is **parsing**: table extraction from PDFs, reading order in multi-column layouts,
OCR quality. That is a solved commercial problem and should never be hand-rolled.

Consequence: the chunker built in Phase 2 will not survive contact with a real document
corpus, and the correct fix is not a better chunker — it is a document-parsing service
(Azure AI Document Intelligence, Content Understanding, or Unstructured) placed in front
of it. Our `HtmlDocumentParser` occupies the position that a parsing service would hold
in a real deployment, which is a reasonable seam but should not be mistaken for
equivalence.

**The .NET ecosystem tax.** LangChain, LlamaIndex, Unstructured, and Docling are all
Python-first. The .NET options (Semantic Kernel, Kernel Memory) are thinner. The
"C# throughout" constraint pushes toward writing chunking by hand more than the broader
ecosystem would.

## Decision

**Build the custom chunker for Phase 2, as planned.** Justified by the Cosmos DB
architecture, by Text Split's lack of a structure-aware mode, and by the learning
objective that is this project's primary purpose.

**Do not present it as the production recommendation.** For an organization adding a RAG
chatbot over internal documents, the default recommendation is Azure AI Search integrated
vectorization — or Foundry IQ if permission-aware knowledge bases are needed — with a
document-parsing service in front for non-HTML sources. The managed path absorbs four of
this roadmap's phases and comes with retry, scheduling, and change detection already
solved.

A credible hybrid worth noting: Cosmos DB is a supported *data source* for AI Search
indexers, so Cosmos as system of record with AI Search as the index is a legitimate
architecture that doesn't require choosing.

## Residual risk accepted

- **Custom code where a free managed feature exists.** Accepted: Cosmos forecloses the
  managed feature, and the learning objective is the point.
- **The chunker will not generalize to messy real-world documents.** Accepted and
  documented above. Mitigated by keeping parsing behind `HtmlDocumentParser`, so a
  parsing service can be substituted without touching the packer.
- **Maintenance burden we would not have with a managed skill.** Accepted, and the reason
  Phase 2 carries a real test suite and Phase 8 carries an eval harness — the
  verification infrastructure is what makes owning this code defensible.

## What would change this decision

- Migrating the vector store to Azure AI Search — then chunking becomes configuration and
  Phase 2's library should be deleted, not ported.
- Ingesting anything other than clean authored HTML — then a parsing service goes in
  front regardless of what happens downstream.
- Needing permission-aware retrieval faster than Phase 11 can deliver it — Foundry IQ
  offers that as a managed capability.
