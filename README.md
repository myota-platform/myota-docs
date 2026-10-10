# MyOTA documentation

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. MPOTA is an optional programme configuration, not the platform
definition. Each programme owns its charter, eligibility, rules, awards,
content, and governance; MyOTA does not copy rules from POTA, MPOTA, or another
initiative.

This repository is the organization’s documentation hub. It does not own
service runtime code. See the [repository map](docs/architecture/repository-map.md) for
service ownership, deployment boundaries, and migration synchronization.

## Project status — current through 11 October 2026

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
size. Phase 2 resumable-upload recovery also passed in isolated CI against the
SeaweedFS image ID deployed to K3s. Phase 3 passed its bounded parser/RSS,
snapshot, and worker-recovery gates in an isolated disposable K3s namespace.
The focused
33-test Phase 3 suite covered all supported file/response formats, maximum-size
geometry, complete snapshots, forced termination/replay at parse, checkpoint,
enrichment, and promotion stages, and graceful JetStream drain. Service CI is
green and the implementation is merged; the image is deployed to the geodata
API and processing worker. Fleet reported 54/54 resources ready and the gateway
health check passed. Phase 4 qualifies the charted geodata API two-to-three pod
range on the current single-node cluster. Phase 5 now has bounded live proof of
API replacement during a 22.5 MB accepted import, CPU-triggered autoscale
up/down, and a read-only two-versus-three replica comparison. That small
10-entity sample showed lower p95 but only a 0.8% throughput increase; a
same-row write test exposed PostgreSQL lock contention. Phase 5 remains open
for broader capacity, independent-row consistency, node/storage failure,
release canary, and rollback qualification. See the [Phase 5 evidence](docs/geodata/evidence/phase5-staged-rollout-2026-10-09.md),
[Phase 3 evidence review](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
and [Phase 4 infrastructure evidence](docs/geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md).
The permanent synthetic
Sevilla set imported 10,000 features in four batches of 2,500. Three batches
promoted 5% each (split between Candidate and Approved); the first had already
been queued for full approval and remains as a documented lifecycle exception.
The measured scale-run catalogue had 2,875 synthetic entities; this is historical
evidence, not the current live inventory. The current catalogue contains ten
entities as verified during the [9 October editor delivery](docs/domain/administration/evidence/entity-catalogue-2026-10-09.md).
The PostGIS plans, bounded
map-load review, retained write-profile summaries, live SeaweedFS correlation,
pre-fixture plans, and evidence limits are recorded in the [Phase 0 evidence
record](docs/geodata/evidence/phase0-production-evidence-2026-10-08.md) and
[representative query review](docs/geodata/evidence/phase0-representative-query-review-2026-10-08.md).

Other outstanding platform work is tracked explicitly in the
[charter gap analysis](docs/governance/charter-gap-analysis.md),
[programme configuration gap analysis](docs/domain/programmes/configuration-gap-analysis.md),
[REST API migration plan](docs/domain/api/rest-consolidation-plan.md), and the
organization’s [profile roadmap](https://github.com/myota-platform/.github/tree/main/profile).

## Documentation map

- Latest implementation deliveries: [changes.md](changes.md), with the earlier
  reconstructed [implementation timeline](docs/history/implementation-timeline.md).
- Browse all subjects in the [documentation index](docs/README.md):
  architecture, domain/API, geodata, operations, observability, governance,
  history, platform policy, security, and development.
- NATS work: [event migration plan](docs/operations/messaging/nats-event-migration-plan.md),
  [Phase 0 evidence inventory](docs/operations/messaging/nats-event-migration-inventory.md),
  [Phase 5 evidence](docs/operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
  and the selected design in [ADR-0008](docs/architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
  Phases 0–5 are complete within the recorded evidence bounds. Helm revision
  193 is deployed and Fleet is Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a. The four legacy Geodata durables
  were retired after final checks; the 24-hour observation was explicitly
  waived and closed early, not reported as a full 24-hour period. Activity and
  replacement Geodata durables, migration 021, and the Interest-retained
  MYOTA_EVENTS stream remain. Phase 6 fact-stream retention is separate.
  The planned NATS broker-monitoring replacement is described in the
  [Surveyor/Grafana roadmap](docs/observability/nats-surveyor-migration.md).
- Track accepted backlog in [To do](docs/to-do/README.md) and active delivery
  and verification in [Work in progress](docs/work-in-progress/README.md).
- **Visual references** — [diagram index](docs/architecture/diagrams/README.md).

Status convention: completed checkboxes in detailed documents mean the listed
work is implemented or verified as stated. Open checkboxes identify work still
in progress or evidence gates not yet met; implementation completion does not
imply scale qualification or production readiness.

NATS migration status on 11 October 2026:

- Phases 0–5 are complete within their documented evidence bounds. Phase 5
  moved all four Geodata work kinds to MYOTA_GEODATA_WORK, applied migration
  021, deployed the Activity idempotency fix, and retired only the four old
  Geodata durables after final checks. The 24-hour wait was explicitly waived;
  this early close is not claimed as a completed 24-hour observation.
- Helm revision 193 remains deployed. Fleet is Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a; all MyOTA Deployments were ready
  at the final check. Activity notification and four target Geodata durables,
  migration markers and recovery data remain.
- The isolated delivery suite had a timing-sensitive immediate ACK-counter
  assertion in each of two runs; focused ACK verification passed, but no clean
  full-suite pass is claimed. Disposable test namespaces were removed.
- MYOTA_EVENTS remains file-backed with Interest retention; Phase 6 owns its
  separate fact-stream retention transition. Off-node recovery remains
  deferred.
- NATS Surveyor and dashboard 16256 are proposed, not deployed. See the
  [monitoring consolidation plan](docs/observability/nats-surveyor-migration.md)
  for the seven-day overlap and retirement gates for the Admin page, broker
  pollers, Grafana dashboard and NATS snapshot table.

See the
[joint review](docs/operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
[Phase 1 completion evidence](docs/operations/messaging/evidence/phase1-completion-2026-10-10.md),
[Phase 2 relay evidence](docs/operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
[Phase 3 consumer evidence](docs/operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
[Phase 4 evidence](docs/operations/messaging/evidence/phase4-activity-work-2026-10-10.md),
[Phase 5 evidence](docs/operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
[Activity notification runbook](docs/operations/messaging/activity-notification-consumer.md),
[Activity work queue runbook](docs/operations/messaging/activity-work-queues.md),
and [current/target diagrams](docs/architecture/diagrams/nats-event-migration.md).


## Source project

The original `ea7klk/mpota` repository remains untouched. Its source and
charter are migration input only; see [migration from MPOTA](docs/governance/migration-from-mpota.md)
and [source inspection](docs/history/source-inspection.md).
