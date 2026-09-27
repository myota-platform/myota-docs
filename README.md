# MyOTA documentation

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. MPOTA is retained as synthetic sample data, not as the platform
definition. Every programme supplies its own charter, eligibility, rules,
awards, content, and governance policy; MyOTA does not copy rules from POTA,
MPOTA, or another initiative.

This repository is the documentation hub for the organization. It does not
own service runtime code. The current repository ownership, migration
synchronization rule, and deployment boundaries are defined in
[`docs/repository-map.md`](docs/repository-map.md).

## Start here

- [Project purpose, motivation, and charter](docs/project-charter.md)
- [Charter-derived gap analysis and delivery sequence](docs/charter-gap-analysis.md)
- [Architecture](docs/architecture.md)
- [REST API consolidation plan](docs/api-rest-consolidation-plan.md)
- [Repository map and ownership boundaries](docs/repository-map.md)
- [Programme configuration gap analysis](docs/programme-configuration-gap-analysis.md)
- [Operations and production-readiness notes](docs/operations.md)
- [Security/threat model](docs/security/threat-model.md)
- [Diagrams](docs/diagrams/)

## Current implementation baseline

The repositories contain a meaningful local vertical slice: amateur-radio
identity with callsigns and SWL participation; programme-owned configuration;
PostgreSQL/PostGIS geodata with candidate/approved/rejected lifecycle;
provenance-aware imports; a relational activity/QSO schema; programme-linked
award execution; a universal public web slice; a separate admin web; and a
durable Colima/Compose deployment using SeaweedFS as the S3-compatible object
store. Activities and awards share the activity API on port 8004.

That baseline is not a claim that the platform is ready for an unrestricted
public launch. The remaining Explorer, participant, governance, integration,
observability, scale, security, and beta-community work is tracked in the
[charter gap analysis](docs/charter-gap-analysis.md) and the organization
[profile roadmap](https://github.com/myota-platform/.github/tree/main/profile).

## Source project

The original `ea7klk/mpota` repository remains untouched. Its source and
charter are migration input only; see
[`docs/migration-from-mpota.md`](docs/migration-from-mpota.md) and
[`docs/source-inspection.md`](docs/source-inspection.md).
