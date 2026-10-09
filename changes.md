# MyOTA changes

Newest deliveries first. Earlier reconstructed service-by-service milestones
remain in the [implementation timeline](docs/history/implementation-timeline.md).

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
