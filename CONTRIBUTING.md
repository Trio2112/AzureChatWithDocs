# Working conventions

This project is built incrementally as a teaching exercise (see
[`docs/roadmap.md`](docs/roadmap.md) for the phase list and
[`docs/plans/semantic-search-learning-project.md`](docs/plans/semantic-search-learning-project.md)
for the original plan and key decisions). These are the process conventions that
apply regardless of which phase is in progress.

## Plan mode at the start of each phase

Each new phase starts with Claude entering plan mode: exploring what's needed,
laying out the approach and any real tradeoffs (e.g. chunking strategy, Cosmos DB
vector index configuration, Container Apps scaling), and getting explicit sign-off
before writing code. This is deliberate — most phases involve architectural decisions
worth understanding and discussing, not just approving.

Small mechanical follow-ups within an already-approved phase (fixing a typo, updating
a status checkbox) don't need a fresh plan-mode round — only the start of a new phase
does.

## Decision records

Each phase's plan document in `docs/plans/` is a deliverable, not scratch paper. It is
committed to the phase branch *before* implementation starts, and it records:

- the decision reached, stated plainly
- the alternatives that were genuinely considered, and why they lost
- the evidence — measurements, version numbers, advisory IDs, benchmark output
- the residual risk knowingly accepted, and what mitigates it

The AngleSharp supply-chain review in `docs/plans/phase-2-chunking.md` is the reference
example. The reasoning is the part that doesn't survive in the code, and in a codebase
where most code is AI-authored it is the more durable artifact of the two — the code
can be regenerated from the decision record far more easily than the decision record
can be reconstructed from the code.

Write plans directly into `docs/plans/<phase-N-short-description>.md` in the repo. Do
not leave them only in a tool's scratch directory.

## Branching

- `main` plus one feature branch per phase, named `phase-N-short-description`.
- Each phase's work happens on its branch and is merged into `main` once the phase is
  reviewed and working.
- No branch protection — solo project.
