# Current system state

**Verified:** 11 October 2026 21:36 UTC  
**Scope:** GitHub `main` repository heads and a read-only inspection of the
MyOTA namespace on the local single-node K3s host. This is a snapshot, not a
production-readiness declaration. Authoritative owner repositories remain the
source of truth; Deploy and Platform copies are synchronized mirrors where the
[repository map](repository-map.md) says so.

## Runtime snapshot

| Area | Verified state |
|---|---|
| Kubernetes | One Ready node, `spainip-k3s`, Kubernetes `v1.35.8+k3s1`. |
| Helm/Fleet | Release `myota`, chart `myota-0.2.14`, revision `261`, status `deployed`. Fleet's MyOTA BundleDeployment is ready at Deploy commit `26d13abaef55f1c545d0fff3b8003f2572bd5483`; all 23 Deployments and 27/27 desired replicas are Ready. |
| MyOTA Deployments | 23/23 report desired replicas ready (27/27 replicas). The admin web is deployed; `myota-web` participant client is not a Deployment in this namespace. |
| Data and storage | Three PostgreSQL targets: `myota_core`, `myota_activity`, and `myota_geo`; PostGIS is used by Geo. SeaweedFS provides S3-compatible object storage. PostgreSQL remains authoritative for domain state and recovery records. |
| Messaging | NATS 2.10, file-backed single replica. See streams below and [ADR-0008](decisions/0008-nats-jetstream-event-and-work-topology.md). |
| Monitoring | NATS Surveyor 0.9.11, Prometheus 2.55.1, Grafana 11.2.2, Alertmanager 0.28.1, OTel Collector 0.135.0, Tempo 2.6.1, and Loki 3.7.8. Eleven NATS dashboards are in Grafana folder `NATS`. |

The Deployment and Fleet observations establish Kubernetes readiness only. They do not establish external availability, multi-node
resilience, capacity, recovery outside this node, or product launch readiness.
The deployed participant web client was absent from the MyOTA namespace check.

## First-party repository heads

These GitHub `main` heads were observed during reconciliation. A source head
does not prove that its image is deployed; deployed image digests are separate.

| Repository | `main` commit |
|---|---|
| [Identity](https://github.com/myota-platform/myota-identity-service) | `846da380ce733863fd7710b1994f008aaaa7fa14` |
| [Programme](https://github.com/myota-platform/myota-programme-service) | `88b12ddd6e31282ae2e0d8822c471eeb77adc954` |
| [Activity](https://github.com/myota-platform/myota-activity-service) | `fe804ddc0b0b5dca3bb0ad0faa1429fdb6ecec18` |
| [Geodata](https://github.com/myota-platform/myota-geodata-service) | `08e416ab746863602c2024f1961c6f66aaec2023` |
| [Operations](https://github.com/myota-platform/myota-operations-service) | `b3851dddc80a2186f29738cafad74532dac64f74` |
| [Contracts](https://github.com/myota-platform/myota-contracts) | `d6445586580a70c155a4797f1122f9b3ddf44eaa` |
| [Deploy](https://github.com/myota-platform/myota-deploy) | `26d13abaef55f1c545d0fff3b8003f2572bd5483` |
| [Platform](https://github.com/myota-platform/myota-platform) | `45437c64ee4f52559b434ede65797e08b8395ac9` |
| [Participant web](https://github.com/myota-platform/myota-web) | `0e6cb4cb03a1bdfac03c288d90af540fba917279` |
| [Admin web](https://github.com/myota-platform/myota-admin-web) | `9d54cc635b7d5411fee6b18d963dc6788a92a43c` |
| [.github](https://github.com/myota-platform/.github) | `6129d75f06ff9fcba713e5f716d9c46b7de76121` |

### First-party images observed in K3s

| Workload | Image digest |
|---|---|
| Identity | `ghcr.io/myota-platform/myota-identity-service@sha256:9bb32a202453319863de357d9dc643bdf6a000e1ae3aa615ffa510e3ea06fa2a` |
| Programme | `ghcr.io/myota-platform/myota-programme-service@sha256:8a89e03ba1cf8a8044e2f99658e6dd7a0d5a099097e7ef7818cdaa9a4d1938c2` |
| Activity API and workers | `ghcr.io/myota-platform/myota-activity-service@sha256:33e7b42f68544b569d621f93bae8fdeec5f696408f0527bc484be6a648673bbc` |
| Geodata API and workers | `ghcr.io/myota-platform/myota-geodata-service@sha256:2f88ec1e48657f751b562ff928c8dd2574289217cda50f086806d5420a51a14a` |
| Operations | `ghcr.io/myota-platform/myota-operations-service@sha256:3649b9977a56ea436b39555fa1b579b619311f11bf6fecfc33b125b302f0c7ca` |
| Admin web | `ghcr.io/myota-platform/myota-admin-web@sha256:95e8279247d61f8395907c57294d4e1102615554766ecff9768ac41f40388fb5` |
| Gateway | `ghcr.io/myota-platform/myota-gateway@sha256:cebd69cf7bfde4dbfe65c16dd1d17e9ac25b8f0c9b894797de0dc392f398b6cf` |
| Shared runtime | `ghcr.io/myota-platform/myota-service@sha256:8050be67423e234c2fad94e5beb6179e34bb692d0b49274d30dc5f7966333f0b` |

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
| NATS event/work migration | Phases 0–6 complete within evidence bounds. Six application DLQ records remain preserved. Seven migration evidence/readiness issues are closed (Operations #1 was superseded); the 24-hour observation was waived and closed early, not completed as a full-day window. Off-node recovery and representative 30-day capacity qualification remain accepted limits. | [Migration plan](../operations/messaging/nats-event-migration-plan.md), [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md), [issue closeout](../operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md) |
| NATS monitoring consolidation | Surveyor, dashboards, capacity settings, and broker-sampler retirement are implemented; single-host broker and Prometheus limits still apply. | [Surveyor plan](../observability/nats-surveyor-migration.md), [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md) |
| SeaweedFS monitoring retirement | Admin page, Operations sampler/API, and sampled-history table are retired; Collector scrape and Grafana dashboard remain. All runtime Deployments are Ready on Helm revision 259 and Fleet's current bundle. | [Deploy issue #8](https://github.com/myota-platform/myota-deploy/issues/8), [monitoring guide](../operations/storage/seaweedfs-admin-status.md) |
| Geodata scale qualification | Phase 5 was administratively closed by owner acceptance on 11 October; broad capacity, independent writes, node/storage failure, canary, rollback, and >3 replicas remain unverified and were accepted for closure. | [Scaling roadmap](../geodata/horizontal-scaling-roadmap.md), [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md), [read-only evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md) |
| Logging and observability | Helm 261 is deployed and all 23 Deployments/27 replicas are Ready. Identity request-to-worker correlation and Identity/Programme metrics coverage are verified; metrics issues #4, #3, and #7 are closed. Programme/Activity/Geodata producer-specific live correlation, authenticated Grafana navigation, Loki privacy/cardinality review, alert/runbook and retention/recovery evidence, and Compose parity remain open. | [Logging roadmap](../observability/logging-implementation.md), [evidence](../observability/evidence/logging-implementation-2026-10-11.md), [metrics verification](../observability/evidence/identity-programme-metrics-2026-10-11.md) |
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
