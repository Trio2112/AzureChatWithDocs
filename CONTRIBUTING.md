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

## Branching

- `main` plus one feature branch per phase, named `phase-N-short-description`.
- Each phase's work happens on its branch and is merged into `main` once the phase is
  reviewed and working.
- No branch protection — solo project.
