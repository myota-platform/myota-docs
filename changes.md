# MyOTA changes

## 11 October 2026 — Phase 5 synthetic validation and observation reset

- Automatic immutable-image digest update produced Helm revision 193. Helm history records it at 22:00:02 UTC on 10 October with status
  deployed; Fleet became
  Ready=True at Deploy commit
  `cfecd655d9c0eee9d19db26725fb11c99366815a` at 22:03:29 UTC. All MyOTA
  Deployments are ready and the migration Job completed.
- A read-only production snapshot at about 22:05 UTC found zero messages/bytes
  in `MYOTA_EVENTS` and `MYOTA_GEODATA_WORK`; all four target work durables
  had exact filters, zero pending/ack-pending/redelivery and one waiting pull.
  The four legacy Geodata durables remained present with zero counters and no
  waiters. No production data or messages were written.
- Injected synthetic stale owner rows into a disposable namespace. Geodata
  recovery recreated four outbox commands once and a second scan emitted none.
  The delivery suite exercised retry, competing consumers, ACK handling,
  redelivery, expiry, shutdown and durable recreation. Two full-suite runs each
  hit one timing-sensitive immediate ACK-counter assertion; focused ACK
  verification passed and all 21 private streams subsequently settled at zero.
  Treat this as isolated behavior evidence, not a clean suite pass.
- Deleted and verified absent the namespace, temporary databases, private
  streams, test fixtures and temporary virtual environment. Revision 193 resets
  the 24-hour observation; it remains open until at least 22:03:29 UTC on
  11 October 2026. Preserve the old durables until the final read-only checks.

## 10 October 2026 — NATS Phase 5 Geodata work cutover and recovery

- Routed all four Geodata work kinds to immutable-image-backed consumers on
  file-backed `MYOTA_GEODATA_WORK`; applied migration 021. PostgreSQL owner
  rows, outbox and recovery indexes remain authoritative. No source rows,
  checkpoints, dead letters or schema needed for recovery were pruned.
- Fixed retry-safe partial Activity/Geodata deletion and transactional Activity
  idempotency. Isolated tests confirmed retry emits exactly one Activity fact.
- Corrected a deployment gap found in live verification: digest values had
  changed rollout annotations while containers still pulled mutable tags.
  Deploy commit [`a68eedd`](https://github.com/myota-platform/myota-deploy/commit/a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109)
  and Platform mirror [`a184baa`](https://github.com/myota-platform/myota-platform/commit/a184baac3f36e0272cbc79107f4b362139de7515)
  now render configured first-party image digests as immutable refs. Fleet first applied Helm revision 191; the latest rollout, revision 192,
  completed at 21:39:22 UTC with Ready=True at Deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a` and
  60/60 resources. Activity, Geodata and shared runtime refs/ImageIDs match the
  configured digests.
- The 154-test Geodata suite, 40-test Activity suite (each with one optional
  skip), 25 relay/topology tests, database refusal/reconnect, same-node NATS
  PVC restart, two-database retry, concurrent idempotency, cancellation race
  and connected expiry/recompletion passed in isolation. The test namespace,
  PVC, streams, fixtures, local API and port-forwards were cleaned up.
- No accepted production Geodata work was available; production data and
  messages were not modified. Keep the four legacy Geodata durables empty
  through the 24-hour observation anchored at revision 192, ending no earlier
  than 21:39:22 UTC on 11 October 2026. A 21:52 UTC read-only sample found
  zero work messages and zero legacy pending/ack-pending/redelivery; it is only
  an early sample, not the gate. The retirement checklist and one-at-a-time CLI
  commands are recorded in the evidence page. Then recheck recovery and remove
  only those four. Activity's notification durable,
  `MYOTA_EVENTS`, migration 021 and authoritative DB rows remain. See the
  [Phase 5 evidence](docs/operations/messaging/evidence/phase5-geodata-work-2026-10-10.md).

## 10 October 2026 — NATS Phase 4 Activity work implementation## 10 October 2026 — NATS Phase 4 Activity work implementation

- Implemented all six selected Activity work commands using atomic job/outbox
  writes, bounded per-kind pull durables, explicit ACK, retry/backoff,
  token-fenced leases, terminal database DLQ records, and audited redrive.
  `NOTIFICATION_SEND` remains excluded and its synthetic rows are removed by
  migration after in-app notification state is corrected.
- Added the guarded, repeatable schema/backfill migration. It preserves the
  `activity_job` status/history table, removes the obsolete DB poller claim
  index, adds lease/DLQ/audit schema, and creates the replacement status index.
- Activity/Contracts/Deploy focused checks passed. Disposable K3s,
  PostgreSQL, and JetStream checks qualified all six subjects/durables,
  backfill gate/rerun, duplicate-safe ACK, stale lease rejection, DLQ/redrive,
  and cleanup. See [Phase 4 evidence](docs/operations/messaging/evidence/phase4-activity-work-2026-10-10.md)
  and [Activity work runbook](docs/operations/messaging/activity-work-queues.md).
- Production cutover completed: `MYOTA_ACTIVITY_WORK` and its six durables are
  live, two workers and three API replicas use the pinned Activity image, and
  compatibility repair is disabled. Migration 007 removed the old claim index
  and synthetic notification jobs while preserving `activity_job` history.
  Prometheus exposes per-kind zero series; the stream and durables have no
  pending or redelivered work. Production had zero selected jobs at cutover, so
  no live handler execution is claimed; all six paths passed isolated
  PostgreSQL/JetStream qualification. Geodata routes and the mixed
  Interest-retained fact stream were not changed. See the [Phase 4 evidence](docs/operations/messaging/evidence/phase4-activity-work-2026-10-10.md)
  and [Activity work runbook](docs/operations/messaging/activity-work-queues.md).

## 10 October 2026 — NATS Phase 3 Activity notification consumer

- Completed the selected committed-fact consumer group: Activity now handles
  19 Identity facts and two Geodata review/status facts through the exact
  `activity-notifications-v1` subject filter. Its projection and idempotency
  record commit together before ACK; poison events are redacted, and redrive is
  audited. Producer routes, Geodata work durables, Activity database-polled
  jobs, and Operations' read-only inspection boundary remain unchanged.
- Deployed on local K3s at Helm revision 170. Live checks found the Activity
  durable healthy with no pending, ack-pending, or redelivered messages; the
  four Geodata work durables and file-backed Interest-retained `MYOTA_EVENTS`
  stream are unchanged. Focused source, deploy, contracts, and isolated broker
  integration checks passed. Test namespaces and repair Job were removed.
- Phase 3 exit criteria are met. Before Phase 4, reconcile the Fleet bundle,
  which still reports `WaitApplied` despite the deployed Helm release and
  60/60 ready resources. See the [Phase 3 evidence report](docs/operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md)
  and [Activity consumer runbook](docs/operations/messaging/activity-notification-consumer.md).

## 10 October 2026 — Phase 1 NATS contract/topology closeout

- Completed ten bounded per-command work schemas with owning-row identity and
  producer/consumer source references; contract tests and the five-service event
  source audit pass. Synchronized registry/schema/docs mirrors to `myota-platform`.
- Added an optional fail-closed Helm pre-upgrade provisioner barrier. It is
  disabled by default and requires explicit migration-gate confirmation. The live
  mixed Interest-retained stream and all runtime producer/consumer paths remain
  unchanged.
- Local K3s evidence covers create/idempotency, drift rejection, local PVC
  restore, bounded replay, and `DiscardNew` pressure rejection. The temporary
  namespace was removed. Volker accepts the conservative caps despite a sample
  shorter than 30 days; off-node recovery is deferred for the single-node scope.
  See [Phase 1 completion evidence](docs/operations/messaging/evidence/phase1-completion-2026-10-10.md).


## 10 October 2026 — NATS Geodata payload contract source pass

- Added additive, source-derived payload schemas for all 27 Geodata facts, with
  classifications for geospatial/reviewer, imported source, and operational
  data. Local contracts tests, audit, schema generation, Ruff, and platform
  mirror comparisons pass. Contracts CI passed in [run 38042564323](https://github.com/myota-platform/myota-contracts/actions/runs/38042564323)
  and platform mirror CI passed in [run 38042622987](https://github.com/myota-platform/myota-platform/actions/runs/38042622987).
  Owner/privacy review remains pending.
- The preprocessed event currently includes internal `_records` and `_status`,
  with `_records` potentially carrying imported source features. Minimize the
  event payload before producer enforcement. No runtime path or deployed
  topology changed. The Phase 0 source audit found no Operations event producer,
  so an Operations payload schema is not applicable unless that service begins
  publishing facts.
- The existing Phase 1 prompt remains in use. See the [Phase 1 evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [plan](docs/operations/messaging/nats-event-migration-plan.md).

Newest deliveries first. Earlier reconstructed service-by-service milestones
remain in the [implementation timeline](docs/history/implementation-timeline.md).

## 10 October 2026 — NATS Activity payload contract source pass

- Added source-derived additive payload schemas for all ten Activity facts and
  classifications for personal activity, import metadata, award configuration,
  and certificate details. Flexible rule/configuration and nested award asset
  shapes remain unconstrained where producer inputs are extensible.
- Local contracts tests (2/2), source audit, schema generation, Ruff format/lint,
  and byte-for-byte platform mirror checks passed. GitHub's connector status
  endpoint returned no combined statuses, so Actions results are not independently
  verified. Commits include contracts `2b8bddaf` and platform mirror `bf6f2727`.
  Joint owner/privacy review, Geodata/Operations payloads, and producer enforcement
  remain open. No runtime or deployed topology changed; the existing Phase 1 prompt
  remains in use.
- See the [Phase 1 evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md).


## 10 October 2026 — NATS Programme payload contract source pass

- Added source-derived additive payload schemas for all 12 Programme facts,
  including content and policy review histories, and marked internal reviewer
  and publisher metadata. Fields the producer accepts without structural
  validation remain unconstrained. Registry tests and CI now cover all 31
  source-derived Identity and Programme fact payloads. Joint owner review and
  payload schemas for Activity, Geodata, and Operations remain open. No runtime
  or deployed topology changed; the existing Phase 1 prompt remains in use.
  Contracts CI passed on `677f8ae`; platform unit, Ruff, and container checks
  passed on `b8c8507`.
  See the [Phase 1 evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [plan](docs/operations/messaging/nats-event-migration-plan.md).

## 10 October 2026 — NATS Identity payload contract source pass

- Added source-derived, additive payload schemas for all 19 Identity facts and
  recorded their data classifications in the registry and generated schemas.
  Authentication events include current email/address fields; the service-token
  event contract excludes the returned token. Contracts CI now runs the schema
  and registry tests. Two tests, Ruff, schema generation, source audit, and
  platform mirror equality checks pass locally. Contracts CI passed on
  `99547de`, and the platform mirror passed unit, Ruff, and container checks on
  `529631c`. Identity owner/privacy review
  and payload schemas for four other producer services remain open. No runtime
  or deployed topology changed. See the [Phase 1 evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [plan](docs/operations/messaging/nats-event-migration-plan.md).

## 10 October 2026 — NATS Phase 1 producer source audit

- Added exact producer source references for all 68 fact types to the
  contracts-owned event registry and a workspace audit that verifies those
  references and rejects undispositioned Python event-like literals. The audit
  covered five authoritative service repositories and found all six legacy Geodata work
  event types classified by the registry. Contract CI checks out those repositories
  and runs the audit; [run 38033202206](https://github.com/myota-platform/myota-contracts/actions/runs/38033202206) passed.
- Synchronized the registry to the platform mirror. Commit `038cd90` passed tests,
  Ruff, and its container check. Two registry tests, Ruff, and local source audit
  passed. Payload schemas remain incomplete; no runtime paths or deployed topology
  changed. See the [Phase 1 evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md).

## 10 October 2026 — NATS Phase 1 durable configuration validation

- Hardened the deploy-owned JetStream provisioner to pin and verify explicit ACK,
  instant replay, bounded pull waiters and pending deliveries, retry settings,
  inherited consumer replicas, durable state storage, and full-payload delivery.
  Normalized NATS server responses that omit false-valued optional settings.
- Four focused tests and Ruff format/lint passed in deploy and its synchronized
  platform mirror. Deploy commit `1076584` passed its Ruff and gateway/service image
  build/publish checks. Platform commit `b047a00` passed Ruff, tests, container
  build, and publish ([deploy checks](https://github.com/myota-platform/myota-deploy/commit/107658467eb708981322649e448832e272daccbb/checks),
  [platform checks](https://github.com/myota-platform/myota-platform/commit/b047a00e15ecc619e3589fffee37a1aa779ff59a/checks)). An isolated host-K3s broker accepted initial provisioning and
  an idempotent rerun for three streams and ten durables; its namespace was
  removed and verified absent. No deployed topology or application path changed.
  Phase 1 remains open. See the [evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md),
  [plan](docs/operations/messaging/nats-event-migration-plan.md), and
  [implementation timeline](docs/history/implementation-timeline.md).

## 9 October 2026 — NATS Phase 1 contract and provisioner preparation

- Added the contracts-owned fact/work registry and envelope schema baseline,
  plus a create-only JetStream provisioner with finite required limits and
  drift rejection. Focused tests and an isolated host-K3s broker test passed;
  the temporary broker namespace was removed.
- Read-only live inspection confirmed the deployed shared `MYOTA_EVENTS` stream
  still uses Interest retention and has no finite byte/message caps. No live
  topology or runtime path changed. Payload schemas, credentials, measured
  limits, restore/replay evidence, and relay-side provisioning removal remain
  open, so Phase 1 is not complete. See the [evidence](docs/operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [plan](docs/operations/messaging/nats-event-migration-plan.md).

## 9 October 2026 — geodata Phase 5 staged rollout evidence

- **K3s rollout and recovery:** Replaced the two Geodata API pods one at a time
  while a checksum-verified, 22.5 MB resumable import was accepted and
  asynchronously processed. Both new pods became Ready, gateway health stayed
  HTTP 200, and exact-run teardown deleted the import, object and upload
  session. No API persistent volume was mounted; stateful services and the
  processing worker were not restarted.
- **Autoscaling and measurements:** A bounded 50-VU same-entity edit profile
  caused the configured 70%-CPU HPA to scale from two to three replicas. After
  load ended, the HPA returned the Deployment to two. A read-only 50-VU sample
  at three replicas recorded 95.50 requests/s and 15.98 ms p95, compared with
  the retained two-replica 94.77 requests/s and 18.76 ms p95. This is a modest
  result on ten entities, not a broader capacity claim. The edit profile
  surfaced a PostgreSQL lock-wait hotspot under deliberate same-row contention;
  independent-row writes still need qualification.
- **Cleanup and deployment:** Three exact-tag write runs report `cleaned=true`;
  the read-only run created no application records. No source, chart, or runtime
  configuration change was required. The bounded K3s rolling replacement and
  HPA actions were the workload tests. A subsequent no-change Helm
  reapply/rollback attempt left Fleet `ErrApplied`; Helm revision 151 remains
  deployed, all MyOTA Deployments and database StatefulSets are Ready, and the
  public API health check returns HTTP 200. The pending transaction records were
  cleared and Fleet's Helm operator restarted, but GitOps readiness still needs
  reconciliation in Rancher. This is not reported as a successful Fleet
  rollout. See [deployment follow-up and Phase 5 evidence](docs/geodata/evidence/phase5-staged-rollout-2026-10-09.md)
  and the [roadmap](docs/geodata/horizontal-scaling-roadmap.md).

## 9 October 2026 — Phase 4 single-node scope confirmed

- **Decision and verification:** Confirmed that the current single-node K3s
  cluster is an acceptable Phase 4 test topology; multi-node failover is not a
  prerequisite for its replica-safety exit criterion. Read-only checks found
  Fleet Ready at deployment commit `a80257d`, geodata API 2/2, processing worker
  1/1, HPA range 2–3, PDB `minAvailable: 1`, one Ready node, and gateway health
  HTTP 200 with valid TLS.
- **Disposition:** The previous live two-to-three-to-two test and connection,
  storage, and queue bounds satisfy Phase 4 within this scope. No application
  or Helm change was needed, so no redeployment was triggered. Phase 5's
  broader performance and stateful/node resilience work remains distinct.
- **Documentation:** Clarified the acceptance boundary in the
  [roadmap](docs/geodata/horizontal-scaling-roadmap.md), added the live-state
  confirmation to [Phase 4 evidence](docs/geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md),
  and recorded this follow-up prompt in the [implementation timeline](docs/history/implementation-timeline.md).

## 9 October 2026 — geodata Phase 4 bounded replica safety

- **Deployment:** Added independent geodata API CPU autoscaling from two to
  three replicas, safe rollout/PDB/topology controls, bounded API and worker
  database pools, a Helm connection-budget guard, resource requests/limits and
  worker replica caps. Local Compose now uses matching bounded pools.
- **Live verification:** On `spainip.es`, temporarily scaled the API from two
  ready pods to three and back to two. Gateway health stayed HTTP 200; database
  usage was 7/100 connections and all five checked JetStream consumers had no
  pending, ack-pending, redelivery or message-age backlog. No test/application
  records were created. Current single-node and single-replica stateful storage
  remain explicit availability limits.
- **Validation/deployment:** Deployment commit
  [`a80257d`](https://github.com/myota-platform/myota-deploy/commit/a80257d36ac1d0046fd25b46bc6e0b1172902ef2)
  was pushed; Helm safety rendering, repository quality and image build
  workflows passed. Fleet reconciled the chart and the API returned to two
  ready replicas. The local Compose command could not be validated because the
  host Docker CLI lacks the Compose subcommand.
- **Status:** Phase 4 is complete only for this documented two-to-three API
  pod envelope. Phase 5 retains sustained capacity, node/storage failure,
  canary and rollback gates. See the [Phase 4 evidence](docs/geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md)
  and [roadmap](docs/geodata/horizontal-scaling-roadmap.md).

## 9 October 2026 — geodata Phase 3 bounded-worker gates closed

- **Geodata service:** Fixed zipped Shapefile streaming so PyShp reads bounded
  scratch files rather than ZIP streams. Added fixed-chunk XML preflight to
  reject DTDs, over-limit feature subtrees, and excessive element counts before
  ElementTree allocation. Added OSM area ring-vertex checks before GeoJSON
  conversion and corrected incremental GeoJSON size accounting so valid large
  geometries are measured without repeatedly counting the full parser path.
- **Failure recovery:** Added isolated forced-process-death/replay checks before
  the first parse checkpoint, after a committed checkpoint, during location
  enrichment before commit, and during promotion before its durable checkpoint.
  Added graceful JetStream shutdown coverage while a handler is active.
- **Evidence:** The focused Phase 3 suite passed **33/33 tests** on
  `spainip.es` using a dedicated disposable K3s namespace, PostGIS `*_tests`
  database, and temporary JetStream broker. No application data or production
  workers were used. RSS remained below 96 MiB for 300k GeoJSON/WFS/ArcGIS
  features, 50k KML/GPX/Shapefile/ParkServe/OSM PBF features, a 250k-vertex
  GeoJSON feature, and a 1k-feature complete-snapshot worker run. Exact per-case
  measurements and recovery assertions are in the [Phase 3 evidence report](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md).
- **Validation/deployment:** Ruff check/format and all service CI jobs passed.
  The merged image build passed and published
  `sha256:04202907b6a0e872f14aa86d9a98c7932e1c10f5b6f29fa47672b7baef4ba275`.
  Deployment commit `54ac990cc156ef85d841ab089284b8c82a1c7685` pins it;
  GitHub Helm rendering and deployment quality passed. Fleet observed the new
  commit and reported all 54 resources ready. Geodata API and import-processing
  worker rollouts succeeded, and the public gateway health endpoint returned
  healthy. The disposable test namespace and fixtures were removed; application
  databases and object storage were not altered.
- **Links:** [Phase 3 roadmap](docs/geodata/horizontal-scaling-roadmap.md#phase-3--isolate-preprocessing-and-promotion-from-api-pods),
  [Phase 3 evidence](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
  [implementation timeline](docs/history/implementation-timeline.md),
  [service merge](https://github.com/myota-platform/myota-geodata-service/commit/cf286931db526ad1c990ed9fa32a0db37a5ff98a),
  [image build](https://github.com/myota-platform/myota-geodata-service/actions/runs/37924977799),
  [deployment commit](https://github.com/myota-platform/myota-deploy/commit/54ac990cc156ef85d841ab089284b8c82a1c7685),
  [Helm validation](https://github.com/myota-platform/myota-deploy/actions/runs/37925422702),
  [service repository](https://github.com/myota-platform/myota-geodata-service).

## 9 October 2026 — geodata Phase 3 parser and recovery follow-up

This is an intermediate evidence snapshot. Its open items were closed by the
later Phase 3 qualification entry above.

- **Geodata service:** Added streaming KML, GPX, zipped Shapefile/ParkServe,
  and OSM PBF decoders alongside incremental GeoJSON. Added feature-size,
  vertex, archive-expansion/member, and Shapefile-record guards. Snapshot
  preprocessing now preflights the 5,000-feature limit and reopens the
  immutable object instead of keeping a second decoded feature list. Worker
  batches now stop at 100 features or 32 MiB serialized source data, whichever
  comes first, while isolating an individually-large but permitted feature.
  Added a disposable JetStream CI integration for ACK-pending release,
  commit-before-ACK redelivery and competing pull consumers. Recovery fixtures
  now read authoritative database rows and do not leak test configuration.
- **Evidence:** Local RSS subprocesses processed 300,000 GeoJSON features at
  23.9 MB peak, and 50,000 each of KML/GPX records at 20.5/20.6 MB. Sixteen
  database-backed recovery/concurrency tests passed against a disposable local
  PostGIS database; its container and attached volume were removed after a
  check found earlier interrupted-run fixtures. The latest full local suite
  collected 137 tests: 116 passed, with 21 isolated-service skips; Ruff passed.
  The isolated PostGIS/JetStream GitHub run passed all 137 tests with no skips;
  worker ACK-pending, commit-before-ACK replay, redelivery and competing
  consumers passed. GeoJSON/KML/GPX peaks were about 61.5 MB in CI, below the
  96 MiB ceiling. The final teardown-cleanup refinement also passed. No live
  application-data writes or test entities were created.
- **Status at this checkpoint (superseded):** worst-case XML/PBF memory, all-format/snapshot/remote-adapter
  RSS, and forced worker termination at parse/enrichment/promotion boundaries.
  Phase 3 remains incomplete because of those parser and worker-stage gaps.
  The final commit image was published as
  `sha256:97cf732f9f14f8e5ff76b665b9a893f78033b7e954db80ad8ca2fe3d5ff74780`.
  Deployment commit
  [`5afb183`](https://github.com/myota-platform/myota-deploy/commit/5afb18379a153ef04ad7cfe310e5efe81d4b102d)
  pins this image for Fleet. Both geodata API and import worker now run this
  digest; https://api.myota.top/healthz returned healthy. Helm rendering and
  repository quality workflows passed. No live import or fixture data was
  written.
- **Links:** [Phase 3 evidence](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
  [horizontal-scaling roadmap](docs/geodata/horizontal-scaling-roadmap.md),
  [geodata service commit 2de096f](https://github.com/myota-platform/myota-geodata-service/commit/2de096f90d3a68092e69c0456246c6a1e0e67d59),
  [passing final CI rerun](https://github.com/myota-platform/myota-geodata-service/actions/runs/37920214726),
  [deployment commit 5afb183](https://github.com/myota-platform/myota-deploy/commit/5afb18379a153ef04ad7cfe310e5efe81d4b102d),
  [Helm render workflow](https://github.com/myota-platform/myota-deploy/actions/runs/37920788467),
  [geodata service README](https://github.com/myota-platform/myota-geodata-service#readme).

## 9 October 2026 — geodata Phase 3 bounded preprocessing (partial at that point)

This earlier checkpoint is retained for timeline context; the later delivery
entry above records the completed bounded parser and worker-recovery gates.

- **Geodata service:** Added streamed object-to-scratch downloads and an
  incremental `ijson` path for uploaded GeoJSON FeatureCollections/arrays.
  Candidate persistence now checkpoints by stable ordinal in configurable
  100-feature windows and evicts committed row projections before advancing.
  Compose and Helm expose the batch-size setting. Replay/cancellation and
  source metadata behavior remain covered by unit regressions.
- **Verification boundary:** Ruff passed; the local suite had 104 passes and
  17 environment-dependent skips (121 collected); the focused import/worker suite passed
  32 tests with its `ijson`-dependent test skipped locally. This host could not
  reach PyPI, so local verification used the parser compatibility fallback.
  The actual `ijson` path passed in the [PostGIS-backed GitHub quality run](https://github.com/myota-platform/myota-geodata-service/actions/runs/37913194172),
  which installed service requirements and passed all 121 tests without skips.
  Whole-document fallbacks remain for KML, GPX, Shapefile and snapshot imports.
  No peak-RSS measurement, worker termination/forced reclaim, or concurrent
  worker proof exists yet; Phase 3 remains open.
- **Evidence:** [Phase 3 implementation and exit review](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
  [horizontal-scaling roadmap](docs/geodata/horizontal-scaling-roadmap.md),
  [geodata service commit 301bc28](https://github.com/myota-platform/myota-geodata-service/commit/301bc28),
  and [deployment commit 890197f](https://github.com/myota-platform/myota-deploy/commit/890197f).
  Service validation passed locally: 104 passed and 17 environment-dependent
  tests skipped; Ruff passed. The streaming-dependency test was skipped locally
  because this host could not reach PyPI. Updated root status, geodata index,
  WIP, To do, and prioritized backlog with the precise partial status.
  [image publication](https://github.com/myota-platform/myota-geodata-service/actions/runs/37913192906)
  and [Helm chart validation](https://github.com/myota-platform/myota-deploy/actions/runs/37913227459)
  passed. Fleet observed deployment commit `890197fa4f00b828a0be3b1b4ab4b645471ba89f`;
  BundleDeployment reached `Ready=True`, Helm revision 118 is deployed, and
  the API and worker are each 1/1 ready. The worker uses batch size 100 and
  the published service image digest recorded in the linked evidence report.
  The gateway health check passed. No production import or worker fault
  injection was performed. At this checkpoint Phase 3 remained open pending
  bounded-path and worker-recovery evidence; those gates are closed by the
  later [Phase 3 qualification entry](docs/history/implementation-timeline.md#9-october-2026--geodata-phase-3-bounded-processing-and-recovery).

## 9 October 2026 — geodata Phase 2 upload recovery gate

- **Geodata service:** Added a repeatable CI failure-injection scenario pinned
  to the exact SeaweedFS image ID observed in the live K3s pod. In an isolated
  test environment, it restarts the API between multipart parts and restarts
  the SeaweedFS container on a disposable persistent volume; verifies resume,
  per-part and whole-object checksums, import metadata/outbox durability,
  idempotent retry, owner isolation and abort cleanup. Production storage was
  not restarted or written. The service README now links the runbook/evidence.
- **Evidence:** [Phase 2 recovery report](docs/geodata/evidence/phase2-upload-recovery-2026-10-09.md);
  [geodata service commit 853fcbc](https://github.com/myota-platform/myota-geodata-service/commit/853fcbc5b30085c8a450ff0accbe32e414348c22);
  [GitHub Actions run 37908154059](https://github.com/myota-platform/myota-geodata-service/actions/runs/37908154059)
  passed the recovery, relational-boundaries, load-harness and image-publishing
  jobs. The follow-up service documentation/image build also passed in
  [run 37908394130](https://github.com/myota-platform/myota-geodata-service/actions/runs/37908394130).
- **Helm rollout:** Digest sync was committed as
  [`myota-deploy` 159861e](https://github.com/myota-platform/myota-deploy/commit/159861e1d77461882f49b7ff44859ce61ebd1845).
  [GitHub Helm render](https://github.com/myota-platform/myota-deploy/actions/runs/37908782124)
  passed. Fleet reconciled the commit with `Ready=True`; Helm revision 115 is
  deployed, the new geodata image is Ready, all 21 deployments are Ready, and
  the public gateway health check succeeded. Detailed image and deployment IDs
  are in the [Phase 2 evidence report](docs/geodata/evidence/phase2-upload-recovery-2026-10-09.md).
- **Roadmap:** Phase 2 is complete for the tested digest. Phase 3 bounded
  parsing/worker-failure proof, Phase 4 infrastructure review, and Phase 5
  broader rollout proof remain open. Updated the geodata index, evidence index,
  work-in-progress, to-do/prioritized backlog and retained Phase 2 prompt to
  distinguish completed recovery from future failure scenarios.

## 9 October 2026 — administration workspace reorganization

- **Admin web:** Permission-aware searchable navigation, guarded direct links,
  consistent headings/refresh/actions, explicit programme scope and task entry
  points. Entity categories join Entities; policies and certificate design are
  distinct. Users, roles and security have separate tabs.
- **Admin forms:** Preserve scoped user-role assignments and clear password
  reset fields after saving; immutable roles/published versions and unprivileged
  award/category edits are read-only. Report failures separately from success;
  unavailable metrics are not fabricated zeros. Map loading is explicitly paged.
- **Compatibility:** Existing API resources, routes, map/import/deletion/award
  workflows, UTC and service data boundaries remain. No schema change.
- **Documentation:** [Workspace guide](docs/domain/administration/navigation-reorganization.md),
  [diagram](docs/architecture/diagrams/admin-workspaces.md) and
  [validation/deployment evidence](docs/domain/administration/evidence/admin-workspaces-2026-10-09.md).

Final delivery: 30 unit tests, 23 browser checks, production build/types and
GitHub Helm rendering passed. Live read-only verification covered 14 pages with
zero browser/API failures or attempted writes. Helm **0.2.12 revision 114** is
deployed, Fleet **1/1 Ready**, and all 21 MyOTA deployments are Ready. See the
evidence page for commits, workflow links, image digest and verification limits.
