# Phase 2 — Chunking strategy (library in `Shared`)

## Context

Phase 1 left us with 10 well-formed HTML documents in `content/` and an empty
`src/Shared` (still `Class1.cs`). Before we can embed anything in Phase 4 or store
vectors in Phase 5, we need to decide how a document becomes the units we actually
embed. That decision is the single biggest lever on retrieval quality in the whole
project, and "chunking and indexing strategies" is directly on the AI-200 syllabus.

**Why chunk at all, for this corpus specifically.** Not for model limits —
`text-embedding-3-small` accepts 8191 tokens and our longest document
(`remote-work-policy.html`) is ~1300. We chunk for *retrieval precision*: a
whole-document vector is the centroid of every topic in the doc, so `pto-policy.html`
and `benefits-overview.html` both read as "generic HR blur" and neither is
convincingly close to *"how many PTO days carry over?"*. A 113-token
`[Carryover and payout]` chunk is almost entirely about carryover, and wins
decisively. Secondarily, it keeps the Phase 7 chat prompt affordable.

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
  `<meta name="category">`, `DocumentId` a stable slug from the path (`hr/pto-policy`).

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

- `ChunkingOptions.cs` — `TargetTokens=350`, `MaxTokens=500`, `MinTokens=80`,
  `OverlapTokens=50`, `EmbeddingModel="text-embedding-3-small"`. Options object from
  day one; these knobs get retuned once Phase 6 shows real retrieval quality.
- `SourceDocument.cs` / `DocumentSection.cs` — parse output records.
- `Chunk.cs` — `Id` (`{DocumentId}#{Ordinal:D4}`), `DocumentId`, `Title`, `Category`,
  `RelativePath`, `HeadingPath`, `Ordinal`, `Text`, `TokenCount`, `ContentHash`.
  `ContentHash` (SHA-256 of `Text`) is deliberate groundwork for Phase 10's
  content-diffing re-vectorization pipeline.
- `ITokenCounter.cs` + `TiktokenTokenCounter.cs` — wraps
  `TiktokenTokenizer.CreateForModel("text-embedding-3-small")`. The interface lets
  tests use a deterministic fake instead of loading the real vocab.
- `HtmlDocumentParser.cs`, `SectionPacker.cs`, `IChunker.cs` + `HtmlChunker.cs`.
- Delete `src/Shared/Class1.cs`.

**Packages** — `AngleSharp` 1.7.1 (Shared), `Microsoft.ML.Tokenizers` 2.0.0 +
`Microsoft.ML.Tokenizers.Data.Cl100kBase` 2.0.0 (Shared).

**New — supply-chain hardening** (first third-party deps enter the repo this phase)

- `src/Directory.Build.props` — `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>`,
  `<NuGetAudit>true</NuGetAudit>`, `<NuGetAuditMode>all</NuGetAuditMode>`,
  `<NuGetAuditLevel>low</NuGetAuditLevel>`. Applied solution-wide rather than per
  project so future phases inherit it.
- `packages.lock.json` per project, committed. Pins the transitive closure by hash;
  CI in Phase 11 can then restore with `--locked-mode` for verifiable builds.
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
stable and ordinal-ordered; same input ⇒ same `ContentHash`.

**New — `src/Shared.Tests/Fixtures/long-policy.html`** — synthetic, well-formatted,
~5000 tokens with deep nesting and an oversized table. This is the only thing in the
repo that exercises the split path, since the real corpus never will.

**Modified — `src/Ingestion/Program.cs`** — CLI that chunks the corpus, writes
`artifacts/chunks.json`, and prints a per-doc summary (chunk count, token min/max/avg).
This is how we verify the phase with zero Azure resources provisioned.
Add `artifacts/` to `.gitignore`.

**Docs** — new `docs/content-conventions.md` (required `<h1>`, `<meta name="category">`,
heading nesting rules, tables must have `<thead>`) so future documents stay chunkable
by construction. Update `docs/roadmap.md` (Phase 2 → ✅). Update `README.md` repo
layout and "Running it" — and fix the existing `src/AzureChatWithDocs.sln` reference,
which is wrong; the file is `.slnx`.

**Branch** — `phase-2-chunking`, per `CONTRIBUTING.md`.

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
4. Point the CLI at `src/Shared.Tests/Fixtures/` and confirm the long doc splits, that
   overlap is visible between consecutive parts of a split section, and that the
   oversized table's parts each carry the header row.
5. Re-run and diff `chunks.json` — byte-identical, confirming IDs and hashes are
   deterministic (a prerequisite for Phase 10).
6. Confirm `packages.lock.json` files are committed and that `--locked-mode` restore
   succeeds from clean (`git clean -xdf` on a scratch clone), proving the build is
   reproducible from pinned hashes.
7. Walk through the results together, then merge to `main`.
