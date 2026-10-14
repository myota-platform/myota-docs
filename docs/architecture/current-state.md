# Current system state

**Production snapshot:** 2026-10-14 11:09 UTC
**Testing snapshot:** 2026-10-14 19:49 UTC
**Scope:** GitHub repository heads and a read-only inspection and controlled
rollout in the production `myota` and isolated `myota-test` namespaces on the
local single-node K3s host. Production and repository heads were checked at
11:09 UTC; Testing was rechecked at 19:49 UTC. This is a snapshot, not a
production-readiness declaration. Authoritative owner repositories remain the
source of truth; Deploy and Platform copies are synchronized mirrors where the
[repository map](repository-map.md) says so.

## Runtime snapshot

| Area | Verified state |
|---|---|
| Kubernetes | One Ready node, `spainip-k3s`, Kubernetes `v1.35.8+k3s1`. |
| Helm/Fleet | Release `myota`, chart `myota-0.2.20`, revision `288`, deployed. MyOTA Fleet GitRepo and BundleDeployment are Ready 71/71 with zero non-ready resources at Deploy `main` commit `ed87c267fa0aa8981de40f1426dd9ace3b36910f`. Migration Job `myota-migrations-288` completed with the immutable unified runner image; all database readiness markers are true at revision 288. |
| Isolated test release | Helm `myota-test` revision `301` is deployed. The Testing Fleet GitRepo points at `myota-platform/myota/testing` commit `e00ba2960595601df03bd71c41f0641b7914197f`; all 24 Deployments and 3 StatefulSets report desired replicas ready, and all 10 database/observability PVCs are Bound. Logs / Failures and Distributed Traces dashboards are provisioned. See [test deployment](../operations/test-deployment.md) for dashboard, image, and query evidence. |
| MyOTA Deployments | 23/23 report desired replicas ready (27/27 replicas), including the namespace-scoped kube-state-metrics Deployment. The Admin web runs image `sha256:52de069757c0f259f7ad22e86309227d6b2558f689029615b47cbf45b7b362c0`; its Name & categories tab places Website and Description after Entity name and Entity categories and saves the full section with one action. The live bundle exposes `Save details`. See [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2). `myota-web` participant client is not a Deployment in this namespace. |
| Data and storage | Three PostgreSQL targets: `myota_core`, `myota_activity`, and `myota_geo`; PostGIS is used by Geo. SeaweedFS provides S3-compatible object storage. PostgreSQL remains authoritative for domain state and recovery records. |
| Messaging | NATS 2.10, file-backed single replica. See streams below and [ADR-0008](decisions/0008-nats-jetstream-event-and-work-topology.md). |
| Monitoring | NATS Surveyor 0.9.11, Prometheus 2.55.1, Grafana 11.2.2, Alertmanager 0.28.1, OTel Collector 0.135.0, Tempo 2.6.1, and Loki 3.7.8. Eleven NATS dashboards are in Grafana folder `NATS`. |

## Monorepo migration status — checked 2026-10-14 16:02 UTC

The new `myota` repository was created from the current `main` snapshots of
the service, web, contract, and deployment sources. No old branches or commit
histories were migrated. Its first integration PR merged to `testing` as
[`a932f2f`](https://github.com/myota-platform/myota/commit/a932f2f3c511dbaebb27315e4c55646dbaec737b).
All ten PR quality checks passed. The merge reran tests on `testing`;
component, contract, Helm, and deployment runtime tests passed, but each of
eight image pushes failed with GHCR `permission_denied: write_package`.
The remediation is tracked in
[myota #53](https://github.com/myota-platform/myota/issues/53). No new
monorepo image digest was published, and no release PR to `main` was opened.

At 16:02 UTC, a read-only Fleet query still showed the production source as
`myota-deploy/main` at `deploy/helm/myota` and the isolated Testing source
as `myota-deploy/testing` at `deploy/helm/myota-test`. No cluster or runtime
resource changed during this repository migration. Existing deployed revision
and image evidence above remains the latest full runtime snapshot; this section
updates repository-transition status only.

All 49 open source issues were migrated with reciprocal links and closed as
migrated. The old source repositories have zero open issues; `myota` has 61 open issues:
11 workstream parents and the 50 migrated task issues. Each parent contains a
linked task checklist and each task issue links back to its parent. The GitHub
connector exposes no native sub-issue write action, so these are documented
issue-body relationships rather than API-backed parent/sub-issue associations. Branch-protection reads for the new
repository returned HTTP 403, and the GitHub connector has no archive write
action. Therefore rulesets are unverified and former repositories remain
unarchived. See the
[transition evidence](evidence/monorepo-transition-2026-10-14.md).

## Identity account and role policy — repository status checked 2026-10-14

Identity account creation and role-grant hardening is in open monorepo PR
[#70](https://github.com/myota-platform/myota/pull/70), based on the testing
branch. It has not merged or deployed. No production or test-cluster data was
changed for this work; the runtime snapshot above remains the latest verified
state.

The implementation candidate gives every new account the least-privilege
PARTICIPANT role, requires identity.accounts.create to create an account, and
requires the separate identity.roles.assign permission for additional roles.
Role grants must have permissions strictly below the actor's current
permissions; actors cannot change their own role assignments. New accounts
remain PENDING_SETUP and the API rejects permanent passwords. Public
registration returns 403 until email verification and abuse controls are
implemented and deployed.

The PR updates apps/identity/identity.py, the synchronized Compose copy
deploy/services/identity.py, and contracts/openapi.yaml. It also recalculates
account permissions from current Identity roles when validating access tokens,
rather than trusting role claims cached at token issuance.

This does not complete the Identity workstream. Durable Identity state still
hydrates from and persists to the service_state JSONB snapshot through
apps/identity/common.py. The relational authentication tables are not yet the
runtime source of truth, so cross-replica uniqueness, revocation, refresh
rotation, lockout, and recovery consistency remain unqualified.
Invitation/credential setup depends on email delivery. Admin role management,
Gateway account-creation coverage, and registration deployment gates also
remain open under issues #14–19 and #31–36.

Focused local evidence for PR #70: Identity unit suite, 20/20 passed; Compose
Identity and vertical-slice suite, 15/15 passed; authorization suite passed
against both the app source and Compose mirror; Ruff checks passed; the
OpenAPI route audit reported zero missing routes and zero duplicate operation
IDs; generated-client checks passed. OpenAPI YAML syntax and a duplicate-key
scan passed. The host lacks Ruby, so the repository's Ruby duplicate-key script
was not run.

## Geodata import review status

On 14 October, a Testing import exposed a permanent source-reference collision
that left an already-confirmed `APPROVED` promotion in `PROCESSING`. Its source
feature contained an OSM `@id`, but an older importer generated a reusable
manual row number. A separate defect also left duplicate evidence missing after
durable reload.

Geodata PR [#21](https://github.com/myota-platform/myota-geodata-service/pull/21)
and Admin Web PR
[#17](https://github.com/myota-platform/myota-admin-web/pull/17) implemented
durable duplicate evidence and the import summary. They merged to `testing` and
were deployed at Helm revision `161`. Geodata issue
[#20](https://github.com/myota-platform/myota-geodata-service/issues/20) and
Admin Web issue
[#16](https://github.com/myota-platform/myota-admin-web/issues/16) are closed.

Geodata issue [#22](https://github.com/myota-platform/myota-geodata-service/issues/22)
and PR [#23](https://github.com/myota-platform/myota-geodata-service/pull/23)
repair legacy `MANUAL:record-N` placeholders from captured source IDs or
import-scoped fallbacks. PR [#24](https://github.com/myota-platform/myota-geodata-service/pull/24)
adds a stable-source reimport regression check. Both PRs passed CI and merged
to `testing`; issue #22 is closed. The image at Helm revision `176` used digest
`sha256:929b31b9f9509c96ed088e158dc3c14e70c95b6b20ab510ca093b861edbb14ba`.

The current import duplicate rule compares against persisted catalogue entities
in `CANDIDATE` or `APPROVED` lifecycle status. It excludes staged candidates
from this or any other import. Geodata issue
[#25](https://github.com/myota-platform/myota-geodata-service/issues/25) and
PR [#26](https://github.com/myota-platform/myota-geodata-service/pull/26)
implement the rule; both the code and image are deployed at Helm revision `191`
with digest
`sha256:c6c8ac57d4590f19bf990ca21282bd1084be0474d3014f094a2a51475ef0eaaf`.
The deployed read-only proximity check returned only `APPROVED` matches. Issue
#25 is closed after the required tests and rollout completed.

A disposable two-candidate test returned one
`WITHIN_50_METERS` match at 15.73 m after durable reload, and its run, candidates,
and manifest were deleted and verified absent. The previously confirmed
promotion was processed through the normal worker and now appears as `APPROVED`
in both catalogue and map queries. Geodata issue [#20](https://github.com/myota-platform/myota-geodata-service/issues/20)
and Admin Web issue [#16](https://github.com/myota-platform/myota-admin-web/issues/16)
are closed after checks passed. Other unreviewed pending records remain for
administrator review.

The five failed promotion queue records observed during the follow-up
investigation were absent on a later read. No candidates were requeued or
approved as part of this fix. The deployed repair was checked directly without
writing test data, and CI exercised promotion while proving the unrelated
source-reference owner remains unchanged.

The Deployment and Fleet observations establish Kubernetes readiness only. They do not establish external availability, multi-node
resilience, capacity, recovery outside this node, or product launch readiness.
The participant web client is absent from the production `myota` namespace and
is deployed in `myota-test`. The test release status is documented separately
from the production release in the [test deployment record](../operations/test-deployment.md).

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

Operations is retired from runtime: no Deployment, Service, Compose workload, API caller, or image reference remains. The Collector reads bounded PostgreSQL metrics through per-database least-privilege roles; Gateway resolves Grafana roles from live Identity data. Both Operations snapshot tables are absent. The GitHub repository was archived on 2026-10-13 and issue [#3](https://github.com/myota-platform/myota-operations-service/issues/3) is closed. Off-node NATS recovery
is explicitly deferred; the local file store is not a backup.

## Work status

| Workstream | Current status | Source of detail |
|---|---|---|
| Testing image digest automation | The scheduled/manual workflow is now in `myota/.github/workflows/sync-testing-image-digests.yml` on `main`; the duplicate `.github` workflow is deleted. It runs on GitHub-hosted Actions runners. Manual run [#38625702955](https://github.com/myota-platform/myota/actions/runs/38625702955) skipped digest updates because `TESTING_DIGEST_APP_ID` and `TESTING_DIGEST_APP_PRIVATE_KEY` are missing from `myota`; digest synchronization is not yet credentialed. | [Test deployment](../operations/test-deployment.md), [myota issue #53](https://github.com/myota-platform/myota/issues/53), [workflow source](https://github.com/myota-platform/myota/blob/main/.github/workflows/sync-testing-image-digests.yml) |
| Cross-repository architecture review | Phases 0–4 complete, including duplicate-YAML-key rejection, handler/mirror check, and route inventory evidence. | [Review](../../REVIEW.md), [Phase 0 evidence](evidence/review-phase-0-2026-10-11.md), [Phases 1–4 evidence](evidence/review-phases-1-4-2026-10-11.md) |
| NATS event/work migration | Phases 0–6 complete within evidence bounds. Six application DLQ records remain preserved. Seven migration evidence/readiness issues are closed (Operations #1 was superseded); the 24-hour observation was waived and closed early, not completed as a full-day window. Off-node recovery and representative 30-day capacity qualification remain accepted limits. | [Migration plan](../operations/messaging/nats-event-migration-plan.md), [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md), [issue closeout](../operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md) |
| NATS monitoring consolidation | Surveyor, dashboards, capacity settings, and broker-sampler retirement are implemented; single-host broker and Prometheus limits still apply. | [Surveyor plan](../observability/nats-surveyor-migration.md), [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md) |
| Operations Service retirement | Runtime, API, Admin caller, and Operations-only schema are retired. Collector metrics and Gateway/Identity Grafana authorization are live. The GitHub repository was archived on 2026-10-13; issue #3 is closed. | [Retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md), [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3) |
| SeaweedFS monitoring retirement | Admin page, sampler/API, and history table are retired; Collector scrape and Grafana dashboard remain. Helm 272 has 23 Deployments and 27 replicas ready. | [Deploy issue #8](https://github.com/myota-platform/myota-deploy/issues/8), [guide](../operations/storage/seaweedfs-admin-status.md) |
| Database baseline and rebuild | Core, Activity, and Geodata baseline/rebuild issues are complete and closed (Deploy #15/#16, Activity #4/#5, Geodata #8/#9). Each rebuild checklist is 7/7. Exact final archives passed full isolated and baseline data-only restores; databases were rebuilt, application grants restored, and all services validated at Helm revision 288. Final archive checksums, catalogs, row counts, API checks, and cleanup are recorded. | [Baseline inventory](../operations/databases/baseline-migration-inventory.md), [rebuild evidence](../operations/databases/database-rebuild-evidence-2026-10-13.md) |
| Geodata scale qualification | Phase 5 was administratively closed by owner acceptance on 11 October; broad capacity, independent writes, node/storage failure, canary, rollback, and >3 replicas remain unverified and were accepted for closure. | [Scaling roadmap](../geodata/horizontal-scaling-roadmap.md), [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md), [read-only evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md) |
| Observability dashboards | The monorepo owns the Grafana definitions. In Testing, Logs / Failures and Distributed Traces are provisioned in the MyOTA folder at `myota-test` Helm revision 301; the bounded Tempo search found 20 traces but no failed or over-one-second samples. K3s workload, application workflow, PostgreSQL/PostGIS, and NATS dashboards remain subject to their dated evidence and stated metric limits. | [Dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md), [Testing verification](../operations/test-deployment.md), [logging runbook](../observability/logging-runbook.md#provisioned-grafana-dashboards) |
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
