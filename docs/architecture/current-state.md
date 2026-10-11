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
| Helm/Fleet | Release `myota`, chart `myota-0.2.14`, revision `249`, status `deployed`. Fleet GitRepo is Ready 1/1 at Deploy commit `12251ff0ccea4c01ac039cc85cd1ee44e29d1b95`; applied and desired deployment IDs match. All 67/67 resources and 23/23 Deployments are Ready. |
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
| [Geodata](https://github.com/myota-platform/myota-geodata-service) | `09022bef0a57b392372100afabbd268db58149bc` (bounded load-test harness only; deployed image unchanged) |
| [Operations](https://github.com/myota-platform/myota-operations-service) | `1b528971b2a60aab1206d51709853df13bec64c7` |
| [Contracts](https://github.com/myota-platform/myota-contracts) | `cce166ab6f7efef053bc395213fb2d9e2f9583de` |
| [Deploy](https://github.com/myota-platform/myota-deploy) | `12251ff0ccea4c01ac039cc85cd1ee44e29d1b95` |
| [Platform](https://github.com/myota-platform/myota-platform) | `eb79df05b6760902e2e88d3cc2c7d9fc8bf8e503` |
| [Participant web](https://github.com/myota-platform/myota-web) | `0e6cb4cb03a1bdfac03c288d90af540fba917279` |
| [Admin web](https://github.com/myota-platform/myota-admin-web) | `9d54cc635b7d5411fee6b18d963dc6788a92a43c` |
| [.github](https://github.com/myota-platform/.github) | `3bfe081b6b39bfc725fdb086c206a4b12c1b978f` |

### First-party images observed in K3s

| Workload | Image digest |
|---|---|
| Identity | `ghcr.io/myota-platform/myota-identity-service@sha256:aaf698323ffa791f1ea27cec257a88bb5f5183bab7ca247fb96cad26097d9cd2` |
| Programme | `ghcr.io/myota-platform/myota-programme-service@sha256:a1d5bd214248c82f41524286a41c82aacb778d36d0f75899421cd60ecc3ec7e2` |
| Activity API and workers | `ghcr.io/myota-platform/myota-activity-service@sha256:557d2a3ad82ecde994cec9793da56ad3cf830e50aaf986a14cfab81f67ccf1b7` |
| Geodata API and workers | `ghcr.io/myota-platform/myota-geodata-service@sha256:7599c09acecf9cbffd8d4f147af04ea2dc2ff95886464d1e08c047ef1daedd58` |
| Operations | `ghcr.io/myota-platform/myota-operations-service@sha256:88078ff3aae873aa820ed40c04916e3b9aa4ac106404c1916179a26aeb07bfc8` |
| Admin web | `ghcr.io/myota-platform/myota-admin-web@sha256:95e8279247d61f8395907c57294d4e1102615554766ecff9768ac41f40388fb5` |
| Gateway | `ghcr.io/myota-platform/myota-gateway@sha256:cebd69cf7bfde4dbfe65c16dd1d17e9ac25b8f0c9b894797de0dc392f398b6cf` |
| Shared runtime | `ghcr.io/myota-platform/myota-service@sha256:b2a71773f4fd4281e3cd55f314ddbd101d5a1a2b22df7a3ac8c0c4cef2ae4ca8` |

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

Operations has no NATS connection and owns bounded read-only PostgreSQL/PostGIS
metrics and Grafana role resolution. SeaweedFS status/history sampling and its
Admin page are retired; its direct Collector scrape and Grafana dashboard remain.
The retired Operations JetStream history table and index are absent. Off-node NATS recovery
is explicitly deferred; the local file store is not a backup.

## Work status

| Workstream | Current status | Source of detail |
|---|---|---|
| Cross-repository architecture review | Phases 0–4 complete, including duplicate-YAML-key rejection, handler/mirror check, and route inventory evidence. | [Review](../../REVIEW.md), [Phase 0 evidence](evidence/review-phase-0-2026-10-11.md), [Phases 1–4 evidence](evidence/review-phases-1-4-2026-10-11.md) |
| NATS event/work migration | Phases 0–6 complete within evidence bounds. Phase 5's 24-hour observation was waived and closed early, not completed as a full-day window. Six application DLQ records remain preserved. | [Migration plan](../operations/messaging/nats-event-migration-plan.md), [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md) |
| NATS monitoring consolidation | Surveyor, dashboards, capacity settings, and broker-sampler retirement are implemented; single-host broker and Prometheus limits still apply. | [Surveyor plan](../observability/nats-surveyor-migration.md), [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md) |
| SeaweedFS monitoring retirement | Admin page, Operations sampler/API, and sampled-history table are retired; Collector scrape and Grafana dashboard remain. All runtime Deployments are Ready; Helm revision 249 is deployed and Fleet is Ready on the desired commit. | [Deploy issue #8](https://github.com/myota-platform/myota-deploy/issues/8), [monitoring guide](../operations/storage/seaweedfs-admin-status.md) |
| Geodata scale qualification | Phase 5 was administratively closed by owner acceptance on 11 October; broad capacity, independent writes, node/storage failure, canary, rollback, and >3 replicas remain unverified and were accepted for closure. | [Scaling roadmap](../geodata/horizontal-scaling-roadmap.md), [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md), [read-only evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md) |
| Logging and observability | Implementation has live evidence; acceptance remains open for authenticated Grafana navigation, request-to-worker correlation, Loki prohibited-field/cardinality review, Compose parity, and Identity/Programme metrics coverage. | [Logging roadmap](../observability/logging-implementation.md), [evidence](../observability/evidence/logging-implementation-2026-10-11.md) |
| Product/domain backlog | Programme configuration, Admin UI internationalization, award-condition modeling, and charter gaps remain planned work. | [To do](../to-do/README.md), [prioritized backlog](../to-do/prioritized-backlog.md) |

## Evidence and method

The deployment snapshot used read-only `k3s kubectl` and Helm inspection with
the explicit K3s kubeconfig. It checked node readiness, Helm release status,
Deployment replica counts, and MyOTA Fleet BundleDeployment conditions and
resource counts. Repository heads came from GitHub `main`; image digests came
from running Pod specifications. SeaweedFS retirement changes runtime deployments and schema through forward-only
migrations; focused checks and live validation will be recorded in the linked
implementation issues after the rollout. This page is a state summary, not the detailed change log.

The [implementation timeline](../history/implementation-timeline.md) records
the prompt and scope of this reconciliation. Dated evidence pages remain
authoritative for their individual checks; where an older dated status differs,
use this newer runtime snapshot as current state while retaining the earlier
result as history.
