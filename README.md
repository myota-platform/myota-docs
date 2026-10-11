# MyOTA

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. MPOTA is an optional programme configuration, not the platform
definition. Each programme owns its charter, eligibility, rules, awards,
content, and governance.

This repository is the documentation hub; service repositories own runtime
code. Start with the [documentation index](docs/README.md), the
[repository ownership map](docs/architecture/repository-map.md), or the
[current system state](docs/architecture/current-state.md).

## Current state

The latest read-only K3s and repository reconciliation is dated 11 October 2026.
It records Helm release 246, Fleet readiness for Deploy commit
`f7d2e65edd5248d70376cf99ccc98c836b7f95e3`, 23/23 MyOTA Deployments Ready,
service and image revisions, current NATS/observability ownership, open work,
and evidence limits. Readiness is not production qualification.

## Work tracking

- [Work in progress](docs/work-in-progress/README.md)
- [To do](docs/to-do/README.md)
- [Prioritized backlog](docs/to-do/prioritized-backlog.md)
- [Cross-repository architecture review](REVIEW.md)

Dated evidence records preserve the state and limits at the time of each
qualification. The current-state page links to them and is the entry point for
the latest verified snapshot.

## Source project

The original `ea7klk/mpota` repository remains untouched. It is migration
input only; see [migration from MPOTA](docs/governance/migration-from-mpota.md)
and [source inspection](docs/history/source-inspection.md).
