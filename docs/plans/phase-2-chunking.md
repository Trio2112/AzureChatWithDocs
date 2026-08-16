# Phase 2 — Chunking strategy (library in `Shared`)

## Context

Phase 1 left us with 10 well-formed HTML documents in `content/` and an empty
`src/Shared` (still `Class1.cs`). Before we can embed anything in Phase 5 or store
vectors in Phase 6, we need to decide how a document becomes the units we actually
embed. That decision is the single biggest lever on retrieval quality in the whole
project, and "chunking and indexing strategies" is directly on the AI-200 syllabus.

**Why chunk at all, for this corpus specifically.** Not for model limits —
`text-embedding-3-small` accepts 8191 tokens and our longest document
(`remote-work-policy.html`) is ~1300. We chunk for *retrieval precision*: a
whole-document vector is the centroid of every topic in the doc, so `pto-policy.html`
and `benefits-overview.html` both read as "generic HR blur" and neither is
convincingly close to *"how many PTO days carry over?"*. A 113-token
`[Carryover and payout]` chunk is almost entirely about carryover, and wins
decisively. Secondarily, it keeps the Phase 9 chat prompt affordable.

**What the corpus measurements told us.** Every `<h2>` section in the current corpus
is 46–158 tokens — nothing comes close to a 500-token cap. So on today's content the
work is *merging* undersized sections, not splitting oversized ones. But future
documents are expected to be **well-formatted but substantially longer**, which makes
the split path real rather than defensive. Both paths get built and tested, and a long
synthetic fixture exercises the split path that today's corpus never triggers.

**Decisions made in planning:**

| Decision | Choice |
|---|---|
| Strategy | Structure-aware: DOM sections, packed by merging + recursive splitting |
| Target / max chunk size | 350 target, 500 max tokens (configurable) |
| Size measurement | Real tokens — `Microsoft.ML.Tokenizers` + `Data.Cl100kBase` |
| Context injection | Heading path prepended to chunk text before embedding |
| Brent's involvement | Review and discussion; Claude implements |
| HTML parser | AngleSharp 1.7.1, kept after supply-chain review (below) |
| Supply chain | Lockfile + `NuGetAudit` + package source mapping, added this phase |
| Access control | `SecurityLabels` on every chunk; **fail closed** — no tag means no access, and the document is rejected at ingest |
| Eval readiness | `ProfileId` in chunk IDs so chunking strategies can be A/B'd without a reindex |

Out of scope: no embeddings, no Azure resources, no Cosmos DB. This phase ends with
inspectable JSON on disk.

### Dependency vetting — AngleSharp

These are the project's first third-party dependencies, so AngleSharp was reviewed
before adoption (data current as of 2026-08-16).

*Why it's needed at all:* .NET has no built-in HTML parser, and `XDocument` can't be
used — the corpus contains `<meta charset="utf-8">`, valid HTML5 but fatal XML.
AngleSharp implements the actual W3C/WHATWG HTML5 parsing algorithm, so implied
`<tbody>` and heading nesting resolve the way a browser would. That correctness is
load-bearing here, because heading structure *is* our chunk boundary logic.

*Standing:* 299.9M downloads, 5.5k stars, MIT, actively maintained since 2013, 215
released versions. Ships a first-class `net10.0` build with **zero package
dependencies** — no transitive supply-chain surface on our TFM. Its nuspec pins the
source commit SHA, so shipped bits are traceable to source.

*Security history:* exactly one advisory ever — **CVE-2026-54570** (medium, mXSS via
MathML `annotation-xml` integration-point bypass), affecting `< 1.5.0`, fixed in 1.5.0.
Nothing in NVD. It is structurally inapplicable to us: it requires parsing *untrusted*
HTML and re-serializing it into a browser, whereas we parse our own files and extract
plain text. The fix shipped (2026-06-06) five weeks before the advisory was published
(2026-07-17) — proper coordinated disclosure. Repo has a `SECURITY.md`.

*Distribution integrity:* NuGet.org versions are immutable — a published version can
never be modified or re-uploaded, only unlisted. Packages are repository-signed by
NuGet.org (DigiCert G4 chain) with the owner identity embedded; there is no separate
author signature, which is the ecosystem norm.

*Residual risks, accepted:* **bus factor 1** — one maintainer (6,244 commits vs. 72 for
the next contributor) who is also sole NuGet owner, with automated publish from GitHub
Actions on push to `main`. The release workflow scopes `NUGET_API_KEY` at workflow
level rather than per-job and pins actions by major tag rather than SHA. It does
correctly use `pull_request` rather than `pull_request_target`, which closes the main
CI-poisoning vector. These risks are mitigated for us by version immutability plus the
lockfile below: what we pin cannot change underneath us.

*Rejected alternative:* HtmlAgilityPack — more total downloads and a corporate owner,
but no `net10.0` target, no commit SHA in its nuspec, and it is a lenient DOM-ish
parser rather than a spec-compliant HTML5 one.

## Approach

Pipeline, all in `src/Shared/Chunking/`:

```
HTML file → HtmlDocumentParser → SourceDocument (ordered sections)
          → SectionPacker      → Chunk[]  (merge small, split large)
          → JSON via Ingestion CLI
```

### 1. Parse — `HtmlDocumentParser` (AngleSharp)

Walk `<body>` in document order, maintaining a heading stack:

- `<h1>`–`<h6>` open a new section and update the heading path (arbitrary depth — do
  **not** hardcode h2/h3; longer future docs will nest deeper).
- Block elements (`p`, `ul`, `ol`, `table`, `pre`) append to the current section as
  normalized plain-text blocks. Lists become `- item` lines; `<code>` keeps its text.
- **Tables** are flattened to pipe-delimited rows with the header row first, and the
  section is flagged `ContainsTable`. This is what keeps `10+ years | 25 days` from
  ever being severed from its column headers.
- **Empty heading containers** (`vpn-setup-guide.html` has a bare
  `<h2>Troubleshooting</h2>` with no body before its `<h3>`s) produce no section of
  their own; the heading stays in the path of its children.
- Document metadata: `Title` from `<h1>` (fallback `<title>`), `Category` from
  `<meta name="category">`, `DocumentId` a stable slug from the path (`hr/pto-policy`),
  and `SecurityLabels` from `<meta name="access">` (comma-separated). The tag is
  **required**; absence is a validation failure, never a grant. See the access-control
  note below.

### 2. Pack — `SectionPacker`

Merge forward through sections in document order, flushing when the next section would
exceed `MaxTokens`; aim for `TargetTokens`. A trailing chunk under `MinTokens` folds
back into the previous chunk when that stays within `MaxTokens` — this is what stops
the 46-token boilerplate `[Questions]` section from becoming its own useless vector.

Oversized single sections (the long-document case) split recursively, cheapest
boundary first:

1. paragraph/block boundaries, greedily packed
2. sentence boundaries, if a single block still exceeds `MaxTokens`
3. hard token-index split via `GetIndexByTokenCount`, only for a pathological
   single sentence
4. tables over the cap split by row, **repeating the header row into each part**

Overlap (`OverlapTokens`, default 50) is applied *only* between parts of a split
section — structure-derived boundaries aren't guesses and don't need insurance.
`GetIndexByTokenCountFromEnd` gives the overlap window, snapped to a sentence boundary.

### 3. Chunk text format

Every chunk is prefixed with its heading context before embedding — the free ~80% of
Anthropic-style contextual retrieval:

```
Paid Time Off (PTO) Policy

## Carryover and payout
Employees may carry over up to 5 unused PTO days into the following calendar year...
```

Title once at the top; each included section contributes its heading path relative to
the title, then its body. A merged chunk simply lists more than one section.

### 4. Key files

**New — `src/Shared/Chunking/`**

- `ChunkingOptions.cs` — `StrategyName`, `TargetTokens=350`, `MaxTokens=500`,
  `MinTokens=80`, `OverlapTokens=50`, `EmbeddingModel="text-embedding-3-small"`, plus a
  derived **`ProfileId`**: a short stable hash of the whole settings object. Any change
  to a knob yields a new `ProfileId`. Options object from day one; these knobs get
  retuned once Phase 8's eval harness gives us measured retrieval quality.
- `SourceDocument.cs` / `DocumentSection.cs` — parse output records.
- `Chunk.cs` — `Id` (**`{ProfileId}:{DocumentId}#{Ordinal:D4}`**), `ProfileId`,
  `DocumentId`, `Title`, `Category`, `RelativePath`, `HeadingPath`, `Ordinal`, `Text`,
  `TokenCount`, `SourceHash`, `ChunkHash`, **`SecurityLabels`**.

**Why the ID and hashes are shaped this way** (added after the roadmap reweighting):

- **`ProfileId` in the chunk ID** lets two chunking configurations be indexed *side by
  side in the same Cosmos container* without collision. Phase 8 can then A/B
  350/500 against, say, 200/300 on identical questions and compare recall@k directly —
  no destructive reindex, no second environment, and the losing profile is deleted by
  filtering on one field. Retrofitting this after documents are embedded would mean
  re-embedding the whole corpus, so it costs nothing now and a lot later.
- **Two hashes, not one.** `SourceHash` (SHA-256 of the source document's normalized
  text) answers *"did the document change?"* — that is what drives Phase 17's
  re-vectorization diffing. `ChunkHash` (SHA-256 of the chunk's own `Text`) answers
  *"did this particular chunk change?"*, so a one-section edit re-embeds one chunk
  instead of the document's worth. Collapsing these into a single hash would make the
  two questions indistinguishable, which is exactly the bug Phase 17 exists to avoid.
- **`SecurityLabels` on every chunk**, denormalized from the source document rather
  than looked up at query time. Phase 11 enforces authorization as a predicate *inside*
  the Cosmos vector query — filtering after retrieval leaks the existence of restricted
  documents and silently degrades answers, because the top-k slots were spent on results
  that then got discarded. An in-query predicate requires the label to live on the
  indexed record. Carrying the field now costs one array per chunk; adding it later
  costs a full re-embed of the corpus.

**Fail closed, and fail loudly.** A document with no `<meta name="access">` tag, or with
an empty one, grants access to **nobody**. Absence of metadata is never a grant — the
alternative fails open, which means the one document somebody forgets to tag is the one
that leaks. There is no implicit `public`; universal readability is an explicit label
like `all-employees` that a human chose to write.

Silent fail-closed is its own trap, though: an untagged document would be indexed,
invisible to every caller, and nobody would know why answers were missing. So the
ingest pipeline **rejects** such a document rather than indexing an unreachable one —
it is skipped, logged with its path and reason, and counted in the CLI summary as
`rejected`. A non-zero rejected count is a visible, actionable signal, and in Phase 4's
CI it fails the build. (Phase 16 revisits this as one case in the broader
malformed-document story; here the rule is simply strict.)

- `ITokenCounter.cs` + `TiktokenTokenCounter.cs` — wraps
  `TiktokenTokenizer.CreateForModel("text-embedding-3-small")`. The interface lets
  tests use a deterministic fake instead of loading the real vocab.
- `HtmlDocumentParser.cs`, `SectionPacker.cs`, `IChunker.cs` + `HtmlChunker.cs`.
- Delete `src/Shared/Class1.cs`.

**Corpus work in this phase:**

- All 10 existing documents need an explicit `<meta name="access" content="...">` tag
  added. Under fail-closed they are currently *all* unreachable, which is the correct
  behavior and the reason this is a Phase 2 task rather than a Phase 11 one.
- Add two or three deliberately restricted documents (e.g. compensation bands, an HR
  investigation summary) with a non-public label. Without them the unrestricted path is
  the only one ever exercised, and Phase 8's eval has nothing to assert a non-leak
  against.

**Packages** — `AngleSharp` 1.7.1 (Shared), `Microsoft.ML.Tokenizers` 2.0.0 +
`Microsoft.ML.Tokenizers.Data.Cl100kBase` 2.0.0 (Shared).

**New — supply-chain hardening** (first third-party deps enter the repo this phase)

- `src/Directory.Build.props` — `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>`,
  `<NuGetAudit>true</NuGetAudit>`, `<NuGetAuditMode>all</NuGetAuditMode>`,
  `<NuGetAuditLevel>low</NuGetAuditLevel>`. Applied solution-wide rather than per
  project so future phases inherit it.
- `packages.lock.json` per project, committed. Pins the transitive closure by hash;
  CI in Phase 4 can then restore with `--locked-mode` for verifiable builds.
- `nuget.config` at repo root — package source mapping binding `AngleSharp` and
  `Microsoft.ML.*` to nuget.org only, so a private or upstream feed added in a later
  phase cannot shadow them.

The audit setting is what would have surfaced CVE-2026-54570 automatically, and it is
directly on-topic for AI-200's secure-deployment material.

**New — `src/Shared.Tests/`** (xUnit, added to `AzureChatWithDocs.slnx`)

Cases: heading-path construction incl. deep nesting; empty heading container dropped;
table kept intact under cap; oversized table split with header repeated; small
sections merged toward target; no chunk exceeds `MaxTokens`; oversized section splits
with correct overlap; overlap text actually appears in both neighbours; chunk IDs
stable and ordinal-ordered; same input ⇒ same `SourceHash`/`ChunkHash`.

Profile/eval-readiness cases: identical options ⇒ identical `ProfileId`; changing any
single knob ⇒ different `ProfileId`; two profiles over the same corpus produce
non-colliding chunk IDs; editing one section changes that chunk's `ChunkHash` and the
document's `SourceHash` but leaves sibling chunks' hashes untouched (the property
Phase 17's incremental reindexing depends on).

Access-control cases, all asserting fail-closed behavior: `<meta name="access">` parsed
into `SecurityLabels`; a document with **no** access tag is rejected, not indexed; a
document with an **empty** access tag is rejected; a rejected document contributes zero
chunks and increments the rejected count; **every** chunk of a restricted document
carries the label, including chunks produced by splitting an oversized section and by
merging adjacent sections — one unlabelled chunk is a leak, so this is asserted
per-chunk, not per-document; and merging never unions labels across documents.

**New — `src/Shared.Tests/Fixtures/long-policy.html`** — synthetic, well-formatted,
~5000 tokens with deep nesting and an oversized table. This is the only thing in the
repo that exercises the split path, since the real corpus never will.

**Modified — `src/Ingestion/Program.cs`** — CLI that chunks the corpus, writes
`artifacts/chunks.json`, and prints a per-doc summary (chunk count, token min/max/avg).
This is how we verify the phase with zero Azure resources provisioned.
Add `artifacts/` to `.gitignore`.

**Docs** — new `docs/content-conventions.md` (required `<h1>`, required
`<meta name="category">`, **required `<meta name="access">`**, heading nesting rules,
tables must have `<thead>`) so future documents stay chunkable by construction. The
access tag is documented as mandatory and fail-closed: omit it and the document is
rejected at ingest rather than published to everyone.

Update `docs/roadmap.md` (Phase 2 → ✅). Update `README.md` repo layout and "Running
it" — and fix the existing `src/AzureChatWithDocs.sln` reference, which is wrong; the
file is `.slnx`.

**Branch** — `phase-2-chunking`, per `CONTRIBUTING.md`.

## Implementation traps

Written for whoever implements this, possibly in a fresh session with no memory of the
planning conversation. Each of these produces code that satisfies the plan as written
and is still wrong.

**1. `ProfileId` and the hashes must use a stable hash function.** `string.GetHashCode()`
is randomized per process in .NET Core and later — it returns a different value on every
run. Using it anywhere near `ProfileId`, `SourceHash` or `ChunkHash` makes chunk IDs
change between runs, breaks the byte-identical re-run check in Verification, and would
silently defeat Phase 17's change detection by marking every document as modified every
time. Use SHA-256 over a canonical, explicitly-ordered serialization of the inputs.

**2. Normalize line endings before hashing.** This repo has `core.autocrlf=true`, no
`.gitattributes`, and CRLF in the index. A hash computed over raw file bytes therefore
differs between a Windows dev machine and Linux CI, which again marks every document as
changed. Normalize to `
` (and trim trailing whitespace) before hashing, and add a
`.gitattributes` normalizing `*.html` so the working tree stops drifting.

**3. Fail closed is the whole point, and the habitual implementation is the bug.** The
reflex when metadata is missing is to default to permissive — `?? "public"`, or treating
an empty label list as "matches everyone". Both are the exact vulnerability this design
exists to prevent. No tag means no access, and the document is *rejected* at ingest. If
a test can be made to pass by defaulting to public, the test is wrong, not the rule.

**4. Security labels must survive the merge path, not just the split path.** Propagating
labels when splitting an oversized section is the obvious case and easy to get right.
Merging several sections into one chunk is the case that gets forgotten — every
resulting chunk must carry the document's labels, and labels must never be unioned
across documents. Assert per-chunk, never per-document: one unlabelled chunk is a leak.

**5. Verify by running, not by reasoning.** Three assumptions in this plan were not
executed and may be wrong: that `NuGetAudit`/`NuGetAuditMode=all` behave as described on
the .NET 10 SDK (they may already be defaults), that `packages.lock.json` cooperates with
the `.slnx` solution format, and that `dotnet new xunit` adds cleanly to `.slnx`. Run
them; if reality differs, adjust and say so in the phase's decision record rather than
working around it silently.

**6. Commit incrementally, in this order.** This phase grew well beyond "write a
chunker" and may not fit one session. Sequence so that stopping early still leaves the
branch coherent: (a) supply-chain config and test project scaffolding, (b) parser +
models, (c) packer with tests, (d) long fixture and split-path tests, (e) corpus
retagging and new restricted documents, (f) CLI, (g) docs. Do not merge to `main`
without the walkthrough in step 8 of Verification.

## Verification

```bash
dotnet restore src/AzureChatWithDocs.slnx --locked-mode   # lockfile is honoured
dotnet build   src/AzureChatWithDocs.slnx                 # audit runs, expect 0 NU1901-1904
dotnet test    src/AzureChatWithDocs.slnx
dotnet run --project src/Ingestion -- chunk --content ./content --out ./artifacts/chunks.json
```

Then confirm by inspection:

1. Summary shows ~2–4 chunks/doc, ~30 chunks total; **no chunk over 500 tokens** and
   none under 80 except a genuinely short final chunk.
2. `pto-policy` chunks are topically coherent — `[Carryover and payout]` is not
   smeared across a boundary, and the accrual table is intact inside one chunk with
   its header row.
3. Every chunk's `Text` opens with its title + heading path.
4. **Fail-closed check:** every chunk carries at least one security label, and the
   summary reports `rejected: 0` once the corpus is tagged. Temporarily strip the
   access tag from one document and confirm it is rejected with a named reason and
   contributes zero chunks — a document that silently produces unreachable chunks is
   the failure this design exists to prevent.
5. Point the CLI at `src/Shared.Tests/Fixtures/` and confirm the long doc splits, that
   overlap is visible between consecutive parts of a split section, and that the
   oversized table's parts each carry the header row.
6. Re-run and diff `chunks.json` — byte-identical, confirming IDs and hashes are
   deterministic (a prerequisite for Phase 17).
7. Confirm `packages.lock.json` files are committed and that `--locked-mode` restore
   succeeds from clean (`git clean -xdf` on a scratch clone), proving the build is
   reproducible from pinned hashes.
8. Walk through the results together, then merge to `main`.
