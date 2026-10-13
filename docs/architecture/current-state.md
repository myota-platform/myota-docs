# Current system state

**Verified:** 13 October 2026 09:37 UTC
**Scope:** GitHub `main` repository heads and a read-only inspection and controlled rollout in the
MyOTA namespace on the local single-node K3s host. This is a snapshot, not a
production-readiness declaration. Authoritative owner repositories remain the
source of truth; Deploy and Platform copies are synchronized mirrors where the
[repository map](repository-map.md) says so.

## Runtime snapshot

| Area | Verified state |
|---|---|
| Kubernetes | One Ready node, `spainip-k3s`, Kubernetes `v1.35.8+k3s1`. |
| Helm/Fleet | Release `myota`, chart `myota-0.2.20`, revision `282` (rollback to revision 281), deployed. Prometheus and Grafana are 1/1; all 23 Deployments / 27 replicas are ready. MyOTA Fleet GitRepo is Ready 1/1 at Deploy `main` commit `9d80741`; its BundleDeployment is Ready 94/94. The migration Job `myota-migrations-281` ran the historical migration chain; the unified runner remains on `testing` and is not yet deployed. |
| MyOTA Deployments | 23/23 report desired replicas ready (27/27 replicas), including the namespace-scoped kube-state-metrics Deployment. The Admin web runs image `sha256:52de069757c0f259f7ad22e86309227d6b2558f689029615b47cbf45b7b362c0`; its Name & categories tab places Website and Description after Entity name and Entity categories and saves the full section with one action. The live bundle exposes `Save details`. See [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2). `myota-web` participant client is not a Deployment in this namespace. |
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
| [Identity](https://github.com/myota-platform/myota-identity-service) | `c563ecc0ee43b8623c4646da3bfcd021b53abc87` |
| [Programme](https://github.com/myota-platform/myota-programme-service) | `88b12ddd6e31282ae2e0d8822c471eeb77adc954` |
| [Activity](https://github.com/myota-platform/myota-activity-service) | `fe804ddc0b0b5dca3bb0ad0faa1429fdb6ecec18` |
| [Geodata](https://github.com/myota-platform/myota-geodata-service) | `08e416ab746863602c2024f1961c6f66aaec2023` |
| [Operations](https://github.com/myota-platform/myota-operations-service) | `5952738366824a1a9cc494fd2e68737553d73f5e` |
| [Contracts](https://github.com/myota-platform/myota-contracts) | `fb564e3fed6ef4482c29a56b9d194332f5804ee7` |
| [Deploy](https://github.com/myota-platform/myota-deploy) | `9d80741f4f9ff637f652a8607264f5815cc3b268` |
| [Platform](https://github.com/myota-platform/myota-platform) | `5555541c8be02891f91afee9c7e0898e9bff428f` |
| [Participant web](https://github.com/myota-platform/myota-web) | `0e6cb4cb03a1bdfac03c288d90af540fba917279` |
| [Admin web](https://github.com/myota-platform/myota-admin-web) | `cb0e9d855daede65a2383ef1a57b52e6f4d13f87` |
| [.github](https://github.com/myota-platform/.github) | `c57e75eb842831efd2a1293f97363da48835e887` |

### First-party images observed in K3s

| Workload | Image digest |
|---|---|
| Identity | `ghcr.io/myota-platform/myota-identity-service@sha256:d94d14ea4d34fc3378fdd574f6b215566647b77f1297b9fb33bc43deb3e699d1` |
| Programme | `ghcr.io/myota-platform/myota-programme-service@sha256:8a89e03ba1cf8a8044e2f99658e6dd7a0d5a099097e7ef7818cdaa9a4d1938c2` |
| Activity API and workers | `ghcr.io/myota-platform/myota-activity-service@sha256:33e7b42f68544b569d621f93bae8fdeec5f696408f0527bc484be6a648673bbc` |
| Geodata API and workers | `ghcr.io/myota-platform/myota-geodata-service@sha256:2f88ec1e48657f751b562ff928c8dd2574289217cda50f086806d5420a51a14a` |
| Admin web | `ghcr.io/myota-platform/myota-admin-web@sha256:23f4466f743d7f41d4a3549510926effe71e797aacc1a67c22aff85a6cafec54` |
| Gateway | `ghcr.io/myota-platform/myota-gateway@sha256:071096c3b3108fcaa23f765eb4530e579721afc35de80c81e3128612b52ce909` |
| Shared runtime | `ghcr.io/myota-platform/myota-service@sha256:c7585d61e11697fa47076ffffde7839c0eb36e3d94262a540bfd0aa636956d34` |

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

Operations is retired from runtime: no Deployment, Service, Compose workload, API caller, or image reference remains. The Collector reads bounded PostgreSQL metrics through per-database least-privilege roles; Gateway resolves Grafana roles from live Identity data. Both Operations snapshot tables are absent. Repository archive remains pending [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3). Off-node NATS recovery
is explicitly deferred; the local file store is not a backup.

## Work status

| Workstream | Current status | Source of detail |
|---|---|---|
| Cross-repository architecture review | Phases 0–4 complete, including duplicate-YAML-key rejection, handler/mirror check, and route inventory evidence. | [Review](../../REVIEW.md), [Phase 0 evidence](evidence/review-phase-0-2026-10-11.md), [Phases 1–4 evidence](evidence/review-phases-1-4-2026-10-11.md) |
| NATS event/work migration | Phases 0–6 complete within evidence bounds. Six application DLQ records remain preserved. Seven migration evidence/readiness issues are closed (Operations #1 was superseded); the 24-hour observation was waived and closed early, not completed as a full-day window. Off-node recovery and representative 30-day capacity qualification remain accepted limits. | [Migration plan](../operations/messaging/nats-event-migration-plan.md), [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md), [issue closeout](../operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md) |
| NATS monitoring consolidation | Surveyor, dashboards, capacity settings, and broker-sampler retirement are implemented; single-host broker and Prometheus limits still apply. | [Surveyor plan](../observability/nats-surveyor-migration.md), [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md) |
| Operations Service retirement | Runtime, API, Admin caller, and Operations-only schema are retired. Collector metrics and Gateway/Identity Grafana authorization are live. Repository archive remains pending because the GitHub connector exposes no archive operation. | [Retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md), [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3) |
| SeaweedFS monitoring retirement | Admin page, sampler/API, and history table are retired; Collector scrape and Grafana dashboard remain. Helm 272 has 23 Deployments and 27 replicas ready. | [Deploy issue #8](https://github.com/myota-platform/myota-deploy/issues/8), [guide](../operations/storage/seaweedfs-admin-status.md) |
| Database baseline and rebuild | Core, Activity, and Geodata baseline sources and synchronized mirrors are on `testing`. The fixed runner passed fresh, repeat, and readiness-failure rehearsals in disposable K3s databases; runner-test databases/jobs were cleaned. The baseline issues are partially checked off (Deploy 5/7, Activity 5/6, Geodata 5/7) and remain open for complete object-to-callsite mapping and focused service integration checks. Activity's historical migration/table inventory is checked off. Each rebuild issue now has 1/7 checks complete for its checksummed preliminary archive and isolated restore rehearsal; archives predate quiescence and are not cutover backups. Live DBs were not modified; the running Helm image still uses historical migrations. | [Baseline inventory](../operations/databases/baseline-migration-inventory.md), [Deploy #15/#16](https://github.com/myota-platform/myota-deploy/issues/15), [Activity #4/#5](https://github.com/myota-platform/myota-activity-service/issues/4), [Geodata #8/#9](https://github.com/myota-platform/myota-geodata-service/issues/8) |
| Geodata scale qualification | Phase 5 was administratively closed by owner acceptance on 11 October; broad capacity, independent writes, node/storage failure, canary, rollback, and >3 replicas remain unverified and were accepted for closure. | [Scaling roadmap](../geodata/horizontal-scaling-roadmap.md), [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md), [read-only evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md) |
| Observability dashboard enhancements | The Deploy-owned K3s workload, application workflow, and PostgreSQL/PostGIS dashboards are deployed; Platform mirrors match. Per-durable JetStream and several PostgreSQL maintenance metrics remain unavailable and are explicitly disclosed. | [Dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md) |
| Logging and observability | Correlation work is closed for Identity, Programme, Activity, and Geodata. Loki has three indexed labels, sampled payload exclusion, live 14-day policy and compactor activity. Prometheus loaded 33 runbook-linked rules; Grafana mounts two alert runbook links and three operational dashboard links and logged successful dashboard provisioning. Authenticated trace/log UI navigation, Compose runtime parity, and representative capacity/recovery remain open. | [Logging roadmap](../observability/logging-implementation.md), [12 October evidence](../observability/evidence/logging-implementation-2026-10-12.md), [metrics verification](../observability/evidence/identity-programme-metrics-2026-10-11.md) |
| Admin geodata details save | The reopened Website/Description issue is complete: one Save details action persists name, categories, Website and Description; the final image is live and the served bundle was checked. | [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2), [editor guide](../domain/administration/entity-catalogue-editor.md) |
| Product/domain backlog | Programme configuration, Admin UI internationalization, award-condition modeling, and charter gaps remain planned work. | [To do](../to-do/README.md), [prioritized backlog](../to-do/prioritized-backlog.md) |

## Evidence and method

The deployment snapshot used read-only `k3s kubectl`, Helm, and Prometheus query inspection with
the explicit K3s kubeconfig. It checked node readiness, Helm release status, Deployment replica counts,
MyOTA Fleet BundleDeployment status, dashboard ConfigMap provisioning, and
live metric samples. Dashboard enhancement query and mirror evidence is in
[the dated record](../observability/evidence/dashboard-enhancements-2026-10-12.md). Repository heads came from GitHub `main`; image digests came
from running Pod specifications. SeaweedFS retirement changes runtime deployments and schema through forward-only
migrations; focused checks and live validation will be recorded in the linked
the [Operations retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md); it records the live scrape and role checks. This page is a state summary, not the detailed change log.

The [implementation timeline](../history/implementation-timeline.md) records
the prompt and scope of this reconciliation. Dated evidence pages remain
authoritative for their individual checks; where an older dated status differs,
use this newer runtime snapshot as current state while retaining the earlier
result as history.
