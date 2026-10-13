# Documentation index

The [repository README](../README.md) is the project introduction. This page is
the single subject index. For latest verified runtime and repository status,
start at [Current system state](architecture/current-state.md).

## System

- [Architecture](architecture/README.md) — ownership, boundaries, decisions,
  diagrams, and current state.
- [Domain and API](domain/README.md) — identity, programmes, administration,
  awards, and API contracts.
- [Operations](operations/README.md) — deployment, storage, messaging, databases, and
  service runbooks.
- [Database migration inventory](operations/databases/baseline-migration-inventory.md) — current baseline ownership, runner evidence, and gated rebuild status.
- [Observability](observability/README.md) — logging, metrics, traces, NATS
  Surveyor, evidence, and operating guidance.
- [Platform policies](platform/README.md) and [security](security/README.md).
- [Development](development/README.md) — local workflows and quality checks.

## Work and history

- [Work in progress](work-in-progress/README.md) — active implementation and
  evidence gates.
- [To do](to-do/README.md) — accepted work not started or awaiting a decision.
- [Prioritized backlog](to-do/prioritized-backlog.md) — recommended sequence,
  not a committed schedule.
- [History](history/README.md) — dated decisions, changes, and source inspection.
- [Cross-repository review](../REVIEW.md) — findings, mitigation, and
  completion evidence.

## Documentation rules

Roadmaps own task scope and checkboxes; dated evidence pages own the results and
limits of a specific check. The [current-state page](architecture/current-state.md)
summarizes the latest verified state and links to those sources. Historical
records remain snapshots and are not rewritten to look current. Diagrams show
current ownership unless explicitly labeled historical or proposed.

Older top-level `docs/*.md` paths are retained as short compatibility pages
where other MyOTA repositories link to them. Subject indexes link to canonical
documents.
