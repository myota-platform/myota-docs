# MyOTA

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. MPOTA is an optional programme configuration, not the platform
definition. Each programme owns its charter, eligibility, rules, awards,
content, and governance.

This repository is the documentation hub. Runtime source is in the
[myota monorepo](https://github.com/myota-platform/myota). The isolated Testing
deployment tracks its `testing` branch; production continues to track the
previous split Deploy repository until production promotion is completed. The
original `ea7klk/mpota` repository remains untouched and is migration input
only.

## Start here

- [Documentation index](docs/README.md) — subject-based navigation and
  documentation conventions.
- [Current system state](docs/architecture/current-state.md) — latest verified
  runtime, repository heads, and evidence limits.
- [Repository ownership map](docs/architecture/repository-map.md) — authoritative
  repositories and synchronized mirrors.
- [Architecture](docs/architecture/README.md) — boundaries, decisions, and
  diagrams.
- [Monorepo and future-scale assessment](docs/architecture/monorepo-and-scalability-assessment.md)
  — accepted consolidation strategy and current migration evidence; service,
  API, and frontend boundaries remain independent.

## Delivery tracking

- [Work in progress](docs/work-in-progress/README.md) — active work and gates.
- [To do](docs/to-do/README.md) — accepted work not currently in progress.
- [Prioritized backlog](docs/to-do/prioritized-backlog.md) — recommended order,
  not a schedule.
- [Implementation timeline](docs/history/implementation-timeline.md) — dated
  prompts, changes, and evidence.
- [Cross-repository review](REVIEW.md) — findings and mitigation status.

The current-state page is the only rolling runtime snapshot. Dated evidence
records preserve what was checked and the limits of each result.

## Source project

The original `ea7klk/mpota` repository remains untouched. See the
[migration context](docs/governance/migration-from-mpota.md) and
[historical source inspection](docs/history/source-inspection.md).
