# Current system state

**Verified:** 11 October 2026  
**Scope:** GitHub `main` repository heads and a read-only inspection of the
MyOTA namespace on the local single-node K3s host. This is a snapshot, not a
production-readiness declaration. Authoritative owner repositories remain the
source of truth; Deploy and Platform copies are synchronized mirrors where the
[repository map](repository-map.md) says so.

## Runtime snapshot

| Area | Verified state |
|---|---|
| Kubernetes | One Ready node, `spainip-k3s`, Kubernetes `v1.35.8+k3s1`. |
| Helm/Fleet | Release `myota`, chart `myota-0.2.14`, revision `246`, status `deployed`. Fleet bundle `myota-deploy-deploy-helm-myota` is Ready, Deployed, and Monitored at Deploy commit `f7d2e65edd5248d70376cf99ccc98c836b7f95e3`; 67/67 resources are ready. |
| MyOTA Deployments | 23/23 report desired replicas ready. The admin web is deployed; `myota-web` participant client is not a Deployment in this namespace. |
| Data and storage | Three PostgreSQL targets: `myota_core`, `myota_activity`, and `myota_geo`; PostGIS is used by Geo. SeaweedFS provides S3-compatible object storage. PostgreSQL remains authoritative for domain state and recovery records. |
| Messaging | NATS 2.10, file-backed single replica. See streams below and [ADR-0008](decisions/0008-nats-jetstream-event-and-work-topology.md). |
| Monitoring | NATS Surveyor 0.9.11, Prometheus 2.55.1, Grafana 11.2.2, Alertmanager 0.28.1, OTel Collector 0.135.0, Tempo 2.6.1, and Loki 3.7.8. Eleven NATS dashboards are in Grafana folder `NATS`. |

The 23/23 Deployment and 67/67 Fleet observations establish Kubernetes
readiness only. They do not establish external availability, multi-node
resilience, capacity, recovery outside this node, or product launch readiness.
The deployed participant web client was absent from the MyOTA namespace check.

## First-party repository heads

These GitHub `main` heads were observed during reconciliation. A source head
does not prove that its image is deployed; deployed image digests are separate.

| Repository | `main` commit |
|---|---|
| [Identity](https://github.com/myota-platform/myota-identity-service) | `a503fbeffdc4d8a3ae9aeeee3607d81e5ea891fc` |
| [Programme](https://github.com/myota-platform/myota-programme-service) | `ae65e0e9b80173e710e1265da804c3bd9cd3cbdc` |
| [Activity](https://github.com/myota-platform/myota-activity-service) | `3095559cac35f3c0c9a8d021c387e298ac0cf95b` |
| [Geodata](https://github.com/myota-platform/myota-geodata-service) | `ad28632723765463fa683ac175cf430045c769f0` |
| [Operations](https://github.com/myota-platform/myota-operations-service) | `91b5c2ffdb7bc82419e556d3dc9604624d218972` |
| [Contracts](https://github.com/myota-platform/myota-contracts) | `36cf244bedaf6071e50d7c5aef973ccd890128c4` |
| [Deploy](https://github.com/myota-platform/myota-deploy) | `f7d2e65edd5248d70376cf99ccc98c836b7f95e3` |
| [Platform](https://github.com/myota-platform/myota-platform) | `2426e4da186aca49cfa00a58a83746f3573cdf66` |
| [Participant web](https://github.com/myota-platform/myota-web) | `0e6cb4cb03a1bdfac03c288d90af540fba917279` |
| [Admin web](https://github.com/myota-platform/myota-admin-web) | `8c63f9ddc2f50091e975c836e5413e2bd6405c26` |
| [.github](https://github.com/myota-platform/.github) | `3bfe081b6b39bfc725fdb086c206a4b12c1b978f` |

### First-party images observed in K3s

| Workload | Image digest |
|---|---|
| Identity | `ghcr.io/myota-platform/myota-identity-service@sha256:aaf698323ffa791f1ea27cec257a88bb5f5183bab7ca247fb96cad26097d9cd2` |
| Programme | `ghcr.io/myota-platform/myota-programme-service@sha256:a1d5bd214248c82f41524286a41c82aacb778d36d0f75899421cd60ecc3ec7e2` |
| Activity API and workers | `ghcr.io/myota-platform/myota-activity-service@sha256:557d2a3ad82ecde994cec9793da56ad3cf830e50aaf986a14cfab81f67ccf1b7` |
| Geodata API and workers | `ghcr.io/myota-platform/myota-geodata-service@sha256:7599c09acecf9cbffd8d4f147af04ea2dc2ff95886464d1e08c047ef1daedd58` |
| Operations | `ghcr.io/myota-platform/myota-operations-service@sha256:c4736458e9fc754d80cd5cf9022421fb91483e032fe701104622189feeb86e8e` |
| Admin web | `ghcr.io/myota-platform/myota-admin-web@sha256:0134f7a6f30edf8aa15ed7f7123c1a2a678767e63f812a59bead3b0414a35023` |
| Gateway | `ghcr.io/myota-platform/myota-gateway@sha256:73a23ff8abdbdc17e012feadedfdc71a9270be7677b47e6f80c27de3189f568c` |
| Shared runtime | `ghcr.io/myota-platform/myota-service@sha256:b1f89ddc7b0530df4d51427104cd87bd08686457fbb2c80a09bdbe816c5161ac` |

## NATS streams and ownership

| Stream | Subjects | Policy and bounded limits | Purpose |
|---|---|---|---|
| `MYOTA_EVENTS` | `myota.events.>` | File, Limits, one replica, 30 days, 1 GiB, 500,000 messages, 1 MiB per message, DiscardNew | Committed domain facts and bounded replay; not an archive. Activity notifications use one durable with 21 exact fact filters. |
| `MYOTA_ACTIVITY_WORK` | `myota.work.activity.>` | File, WorkQueue, one replica, 30 days, 1 GiB, 250,000 messages, 1 MiB per message, DiscardNew | Six selected Activity commands; six independent durable work filters. |
| `MYOTA_GEODATA_WORK` | `myota.work.geodata.>` | File, WorkQueue, one replica, 30 days, 3 GiB, 500,000 messages, 1 MiB per message, DiscardNew | Four Geodata work commands and four durable consumers. Four legacy Geodata durables have been retired. |

Three database-specific relays publish from Core, Activity, and Geo outboxes.
Facts record committed domain outcomes; work queues distribute competing-
consumer jobs. Scheduled maintenance and database reconciliation remain
scheduler/recovery paths where the database is the source of truth. Six
unresolved Geodata application DLQ records (three
`geodata.import.queued.v1`, three `geodata.import.recovered.v1`) remain
preserved for investigation and are not counted as successful replay.

Operations has no NATS connection and owns read-only storage status/history and
Grafana role resolution. Surveyor owns broker monitoring. The retired
Operations JetStream history table and index are absent. Off-node NATS recovery
is explicitly deferred; the local file store is not a backup.

## Work status

| Workstream | Current status | Source of detail |
|---|---|---|
| Cross-repository architecture review | Phases 0–4 complete, including duplicate-YAML-key rejection, handler/mirror check, and route inventory evidence. | [Review](../../REVIEW.md), [Phase 0 evidence](evidence/review-phase-0-2026-10-11.md), [Phases 1–4 evidence](evidence/review-phases-1-4-2026-10-11.md) |
| NATS event/work migration | Phases 0–6 complete within evidence bounds. Phase 5's 24-hour observation was waived and closed early, not completed as a full-day window. Six application DLQ records remain preserved. | [Migration plan](../operations/messaging/nats-event-migration-plan.md), [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md) |
| NATS monitoring consolidation | Surveyor, dashboards, capacity settings, and Operations retirement are implemented; single-host broker and Prometheus limits still apply. | [Surveyor plan](../observability/nats-surveyor-migration.md), [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md) |
| Geodata scale qualification | Phase 5 remains open: broader capacity, independent-row consistency, node/storage failure, canary, and rollback gates remain. | [Scaling roadmap](../geodata/horizontal-scaling-roadmap.md) |
| Logging and observability | Implementation has live evidence; acceptance remains open for authenticated Grafana navigation, request-to-worker correlation, Loki prohibited-field/cardinality review, Compose parity, and Identity/Programme metrics coverage. | [Logging roadmap](../observability/logging-implementation.md), [evidence](../observability/evidence/logging-implementation-2026-10-11.md) |
| Product/domain backlog | Programme configuration, Admin UI internationalization, award-condition modeling, and charter gaps remain planned work. | [To do](../to-do/README.md), [prioritized backlog](../to-do/prioritized-backlog.md) |

## Evidence and method

The deployment snapshot used read-only `k3s kubectl` and Helm inspection with
the explicit K3s kubeconfig. It checked node readiness, Helm release status,
Deployment replica counts, and MyOTA Fleet BundleDeployment conditions and
resource counts. Repository heads came from GitHub `main`; image digests came
from running Pod specifications. No runtime code, cluster resource, or test
data was changed for this documentation update. No tests were run.

The [implementation timeline](../history/implementation-timeline.md) records
the prompt and scope of this reconciliation. Dated evidence pages remain
authoritative for their individual checks; where an older dated status differs,
use this newer runtime snapshot as current state while retaining the earlier
result as history.
