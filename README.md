# MyOTA documentation

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. MPOTA is an optional programme configuration, not the platform
definition. Each programme owns its charter, eligibility, rules, awards,
content, and governance; MyOTA does not copy rules from POTA, MPOTA, or another
initiative.

This repository is the organization’s documentation hub. It does not own
service runtime code. See the [repository map](docs/repository-map.md) for
service ownership, deployment boundaries, and migration synchronization.

## Project status — 8 October 2026

The organization has a working multi-service vertical slice: radio-aware
identity, programme configuration, relational PostgreSQL/PostGIS geodata,
candidate review and provenance-aware imports, activity/QSO and award
primitives, universal participant/admin web clients, JetStream workers, and
SeaweedFS-backed object storage. The local stack uses Compose/Colima; the
provisional-production stack is deployed to K3s through Fleet/Helm. Prometheus,
Grafana, Alertmanager, and OpenTelemetry are deployed; Identity and Programme
metrics currently have a documented live `/metrics` scrape gap.

Implemented functionality is not the same as production qualification. The
Phase 0 geodata baseline/evidence gate is complete at the measured catalogue
size; broader geodata scale qualification remains open. The permanent synthetic
Sevilla set imported 10,000 features in four batches of 2,500. Three batches
promoted 5% each (split between Candidate and Approved); the first had already
been queued for full approval and remains as a documented lifecycle exception.
The final catalogue has 2,875 synthetic entities. The PostGIS plans, bounded
map-load review, retained write-profile summaries, live SeaweedFS correlation,
pre-fixture plans, and evidence limits are recorded in the [Phase 0 evidence
record](docs/geodata-phase0-production-evidence-2026-10-08.md) and
[representative query review](docs/geodata-phase0-representative-query-review-2026-10-08.md).

Other outstanding platform work is tracked explicitly in the
[charter gap analysis](docs/charter-gap-analysis.md),
[programme configuration gap analysis](docs/programme-configuration-gap-analysis.md),
[REST API migration plan](docs/api-rest-consolidation-plan.md), and the
organization’s [profile roadmap](https://github.com/myota-platform/.github/tree/main/profile).

## Documentation map

- **Purpose and status** — [Detailed hierarchical documentation index](docs/README.md),
  [project charter](docs/project-charter.md), [charter gap analysis](docs/charter-gap-analysis.md).
- **Architecture and decisions** — [architecture](docs/architecture.md),
  [repository map](docs/repository-map.md), [architecture decision records](docs/adr/README.md).
- **Domain and API** — [REST consolidation plan](docs/api-rest-consolidation-plan.md),
  [programme configuration gaps](docs/programme-configuration-gap-analysis.md),
  [activity and awards](docs/awards-and-programme-execution.md),
  [entity categories](docs/entity-categories.md), [identity and security](docs/identity-security.md).
- **Operations and assurance** — [operations](docs/operations.md),
  [observability](docs/observability.md), [threat model](docs/security/README.md),
  [Python quality checks](docs/development/README.md).
- **Geodata scale work** — [scaling roadmap](docs/geodata-horizontal-scaling-roadmap.md),
  [load/query runbook](docs/geodata-load-test-and-query-evidence.md),
  [Phase 0 evidence](docs/geodata-phase0-production-evidence-2026-10-08.md),
  [representative query review](docs/geodata-phase0-representative-query-review-2026-10-08.md).
- **Visual references** — [diagram index](docs/diagrams/README.md).

Status convention: completed checkboxes in detailed documents mean the listed
work is implemented or verified as stated. Open checkboxes identify work still
in progress or evidence gates not yet met; implementation completion does not
imply scale qualification or production readiness.

## Source project

The original `ea7klk/mpota` repository remains untouched. Its source and
charter are migration input only; see [migration from MPOTA](docs/migration-from-mpota.md)
and [source inspection](docs/source-inspection.md).
