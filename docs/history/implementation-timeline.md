# MyOTA implementation timeline

This timeline reconstructs the major implementation milestones from the local
Git histories of the service, client, contract, deployment, and integration
repositories available on 8 October 2026, cross-checked against the current
architecture and operations documentation. It is intentionally a concise
history of meaningful system changes, not a complete commit-by-commit changelog.
Dates are repository commit dates. A commit link points to a representative
change; the linked repository history contains related follow-up fixes.

The architecture evolved quickly during this period. Earlier entries describe
the state at that point in time; later entries may replace an earlier design.
The [repository map](../architecture/repository-map.md) and [architecture](../architecture/overview.md)
describe current ownership and are authoritative for the present-day system.

## 9 October 2026 — programme/award editing, UTC, catalogue and observability

- **Admin workspace reorganization:** Shared permission-aware navigation,
  page headings, refresh and scope controls; searchable sidebar and direct
  task entry points; Users/Roles/Security tabs; scoped role-grant preservation;
  read-only published/built-in records and clear failure feedback. Existing
  catalogue, map, import, deletion, award and UTC flows remain. See the
  [workspace guide](../domain/administration/navigation-reorganization.md) and
  [delivery evidence](../domain/administration/evidence/admin-workspaces-2026-10-09.md).
  Final UI delivery passed 30 unit and 23 browser tests; the 14 live pages passed
  read-only checks. Helm **0.2.12 revision 114** is deployed at
  [digest commit `33ba50d`](https://github.com/myota-platform/myota-deploy/commit/33ba50d),
  Fleet **1/1 Ready**, with all 21 deployments Ready. No live mutation test was
  performed; fixture saves and permission checks are documented separately.

- **UTC throughout MyOTA:** Established the UTC policy across operational
  timestamps, publication dates, UI displays and Grafana. Replaced local-time
  effective-date handling in awards, policies and content with explicit UTC
  inputs that preserve unchanged instants. Added server-side normalization for
  programme and award publication/draft dates, explicit UTC NATS/storage status
  displays, UTC dashboard/default settings and retention CronJob schedules.
  Three live databases already use UTC; historical data and host-wide settings
  are not rewritten. Non-UTC browser/runtime regressions cover the policy.
  See the [UTC runbook](../platform/utc-time-policy.md) and
  [delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md).
  Final Helm **0.2.12 revision 111** is deployed, Fleet **1/1 Ready** at
  [digest commit `4a5b650`](https://github.com/myota-platform/myota-deploy/commit/4a5b650),
  and all **21 deployments** are Ready. Live checks confirmed both programme
  details, all four PDF paper/orientation combinations, UTC UI controls,
  Grafana's UTC default and all five UTC dashboard settings. Eleven browser
  flows, 26 frontend unit tests, activity/programme/runtime regressions,
  contract checks and GitHub Helm rendering passed. No existing live policy
  or award was modified; live artwork uploads/draft writes were not exercised.

- **Programme and award editing:** Fixed reactive-copy failures and selection
  races that left programme identifier/name fields empty, aligned form fields
  and preserved programme-owned metadata on save. Restored six default award
  fields, fetched full award details for editing and preserved legacy layouts.
  Effective-date edits use UTC display time and preserve the exact UTC instant
  of unchanged timestamps instead of shifting it on save.
  Added named PNG/JPEG background and signature uploads through authenticated
  binary content resources, database-backed asset selectors and live artwork
  in the placement canvas. Mock-data preview PDFs open in another window without
  saving or issuing; the shared renderer honours A4/Letter portrait/landscape.
  Preview size/concurrency and image byte/pixel limits bound resource usage.
  Canonical contracts, clients and deployment/integration mirrors are updated.
  See the [designer guide](../domain/awards/admin-designer.md) and
  [delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md).

- **Entity catalogue UX:** Replaced selection-to-scroll editing with a focused
  native dialog and sections for name/categories, location, geometry, immutable
  source comparison and audit/deletion. Previous/next navigation retains the
  loaded catalogue page and independent batch selection. Partial saves preserve
  other drafts, refresh revisions/audit and do not scroll the page. Unsaved
  changes have discard guards; geometry has explicit vertex editing/replacement
  drawing, container resize handling and retired restrictions. Review decisions
  stay inline on the separate review page; single/bulk deletion keeps the
  durable job/impact/confirmation/polling workflow. Corrected the management
  candidate-creation button and per-field location suggestion trees. Added
  browser regression checks, retained CI screenshots, developer instructions,
  [editor guide](../domain/administration/entity-catalogue-editor.md) and
  [navigation diagram](../architecture/diagrams/entity-catalogue-editor.md). Local types,
  24 unit tests, five isolated desktop/mobile browser flows and production
  build passed; the browser tests block public tiles and do not mutate live data.
  Implementation: [admin `5ac4534`](https://github.com/myota-platform/myota-admin-web/commit/5ac4534).
  Browser/CI checks: [admin `2d497e2`](https://github.com/myota-platform/myota-admin-web/commit/2d497e2).
  Follow-up [map reveal/candidate drawing fix `d36a702`](https://github.com/myota-platform/myota-admin-web/commit/d36a702)
  and [read-only live tool `780dad8`](https://github.com/myota-platform/myota-admin-web/commit/780dad8),
  with [origin-confined verification `2a63bf0`](https://github.com/myota-platform/myota-admin-web/commit/2a63bf0).
  The deployed ingress and catalogue/resource/audit APIs returned 200 from K3s,
  and the served asset contains the new editor. Desktop live-browser checks
  were blocked by a local DNS-filter redirect; no live entity was changed.
  Final Fleet/Helm release **103** (`myota-0.2.11`) is deployed at
  [digest commit `1909bde`](https://github.com/myota-platform/myota-deploy/commit/1909bde),
  Fleet 1/1 ready, with all 21 deployments at their desired Ready replicas.
  See [delivery evidence and verification limits](../domain/administration/evidence/entity-catalogue-2026-10-09.md).

- **Storage visibility and Grafana editing:** Added a SeaweedFS Admin UI page
  at `/object-storage`, backed by operations-owned health/exporter snapshots
  and seven-day paged history in `myota_core`. Bucket gauges, filesystem
  capacity and cumulative S3 counters carry explicit unknown and stale states.
  Added a live-identity-validated Grafana session resource and trusted proxy
  headers: GLOBAL_OPERATOR/GLOBAL_ADMIN receive individual Editor identities;
  authorized readers receive Viewer. Provisioned dashboards permit UI saves,
  default to the last 30 minutes and refresh every 30 seconds. Contracts,
  clients, migration mirrors, Compose and Helm are synchronized. See the
  [storage page/access runbook](../operations/storage/seaweedfs-admin-status.md) and
  [sanitized delivery evidence](../observability/evidence/2026-10-09.md).
  Helm chart 0.2.11 revision 97 is deployed; Fleet is 1/1 ready at image-digest
  commit `fb5b655`, and all 21 deployments reached their desired Ready replicas.
  Live checks verified fresh storage samples/history, real storage-health
  metrics, individual Editor identities with spoofed role headers ignored,
  all five dashboard defaults, anonymous denial, and creation/removal of a
  temporary dashboard with a timeseries panel. Local tests, CI, types, build,
  contract reconciliation and GitHub Helm rendering passed. A signed-in
  browser was unavailable for a visual page review; this remains unperformed.

## 8 October 2026 — scale evidence and deployment observability

- **JetStream event retention:** Changed the shared `MYOTA_EVENTS` stream from
  `Limits` to `Interest` retention after ensuring the five required durable
  consumers exist with their current subject filters and explicit acknowledgments.
  This removes an event only after every matching durable consumer acknowledges
  it, while leaving unconsumed work pending; the existing 30-day maximum age
  remains a safety bound. The pre-rollout cluster inspection found 20,885
  retained messages and zero pending or acknowledgment-pending messages across
  those five consumers. An isolated NATS 2.10 integration test verified that
  switching retention removes an already-acknowledged message, preserves a
  pending message until its ACK, and retains a newly published unconsumed
  message. See [JetStream retention operations](../operations/runbooks.md#jetstream-event-retention)
  and the [event contract](https://github.com/myota-platform/myota-contracts/blob/main/contracts/events.md).
  After rollout, the live stream reported `interest`, zero stored messages,
  and zero pending/ack-pending across all five durables, confirming the 20,885
  pre-rollout messages had already been acknowledged and reclaimed. The
  outbox image digest is
  `sha256:493363b21a34384c17fa2b25de5b14d980e88486693c1cb5c8fa5193e2460253`;
  Fleet recorded it in [deployment commit
  `7904ac5`](https://github.com/myota-platform/myota-deploy/commit/7904ac5).
  Helm revision 94 is `deployed`, Fleet is 1/1 ready, and the core, geo, and
  activity outbox deployments are Ready. Platform unit/integration tests,
  deployment tests, CI lint, and image builds passed after CI was updated to
  install the declared service requirements.

- **Admin web:** Fixed stale location metadata after successful asynchronous
  enrichment. Live PostGIS records showed five recent requests had already
  completed with provider status `ENRICHED` and country/city fields persisted;
  the Admin UI refreshed only once immediately after queueing and did not fetch
  the later worker result. Entity Management now polls the selected queued
  entity every two seconds, merges the latest resource into the detail and
  visible list, and stops on completion/failure or selection change. See the
  [enrichment lifecycle](../geodata/location-enrichment.md) and
  [Admin web implementation](https://github.com/myota-platform/myota-admin-web/commit/73019a7).
  The published image was synchronized in [deployment commit
  `29e7a26`](https://github.com/myota-platform/myota-deploy/commit/29e7a26);
  Fleet deployed Helm revision 93 and the Admin web pod became Ready.

- **Geodata provider credential activation:** Provisioned the dedicated
  `myota-geodata-enrichment` Secret (`api-key`) and restarted the API and
  processing worker so both load the Secret-backed environment variable. The
  secret reference was already present in Helm; the running pods had started
  before the Secret was provisioned and therefore retained an empty value.
  Verification confirmed the credential is available in both runtimes without
  printing it, the location-enrichment JetStream consumer is subscribed, and a
  sanitized BigDataCloud lookup returned `ENRICHED` for a Sevilla-area
  coordinate. Earlier failed jobs were already acknowledged; users must submit
  a fresh **Update missing location data** request for entities still missing
  metadata. The Helm chart now provides a
  `geodataPipeline.locationEnrichment.rolloutRevision` value on both geodata
  pod templates; increment it after future Secret creation/rotation so Fleet
  performs the restart. See the [enrichment recovery
  guide](../geodata/location-enrichment.md#durable-coordinates-and-troubleshooting)
  and [deployment instructions](https://github.com/myota-platform/myota-deploy/blob/main/deploy/helm/myota/DEPLOYMENT.md).

- **Geodata service:** Fixed location enrichment jobs that were published and
  consumed but failed as `SKIPPED_NO_CENTROID`. The database row projection had
  omitted the separate PostGIS centroid column, even though affected entities
  retained both geometry and centroid in PostGIS. Durable reads now reconstruct
  `{lon, lat}` from that column, falling back to `ST_Centroid(geom)` for legacy
  rows. Live diagnosis confirmed five request events with no outbox backlog,
  four distinct affected entities, and a subscribed JetStream consumer. The
  failed events were acknowledged, so those entities require a manual retry
  after deployment. K3s rolled the API and worker to image digest
  `sha256:4547e6f39091b7027c42758ad3e164e582f287f133faf3bf0a5758b18cc56775`,
  and the public gateway health check passed. The deployed pod references the
  optional `myota-geodata-enrichment/api-key` Secret field, whose Secret is
  absent under its configured dedicated Secret name; the later credential
  provisioning and pod restart are recorded in the follow-up entry above. See
  the [location enrichment guide and recovery
  notes](../geodata/location-enrichment.md#durable-coordinates-and-troubleshooting)
  and the [service fix](https://github.com/myota-platform/myota-geodata-service/commit/0b3655e).

- **Geodata service, Admin web, contracts and deployment:** Moved reverse
  geocoding out of import preprocessing and request handling into the durable
  geodata worker lifecycle. Enrichment is requested after entity
  materialization when location fields are missing, after geometry changes, or
  when manually managed values are released. The worker verifies the current
  request ID and geometry hash before saving, keeps manual values and their
  codes authoritative, and avoids repeating provider calls after an
  acknowledgement-loss replay. Entity Management now offers an explicit
  “Update missing location data” action; the API request is recorded in the
  transactional outbox and consumed by `geodata-location-enrichment-v1`.
  See the [lifecycle and retry guide](../geodata/location-enrichment.md),
  [REST endpoint contract and consolidation record](../domain/api/rest-consolidation-plan.md),
  and [JetStream event contract](https://github.com/myota-platform/myota-contracts/blob/main/contracts/events.md).
  Commits: [geodata service](https://github.com/myota-platform/myota-geodata-service/commit/81511a9),
  [Admin web](https://github.com/myota-platform/myota-admin-web/commit/20844c6),
  [contracts](https://github.com/myota-platform/myota-contracts/commit/ae99431),
  [platform contract mirror](https://github.com/myota-platform/myota-platform/commit/be5cd3b),
  [Compose/Helm worker configuration](https://github.com/myota-platform/myota-deploy/commit/75146ce), and
  [Helm render regression check](https://github.com/myota-platform/myota-deploy/commit/3187394).
  The deployed GHCR digests are recorded in the [Fleet rollout commit](https://github.com/myota-platform/myota-deploy/commit/e71915c).
  The images passed [geodata CI/build](https://github.com/myota-platform/myota-geodata-service/actions/runs/37814212866),
  [Admin web build](https://github.com/myota-platform/myota-admin-web/actions/runs/37814213107),
  [contract freeze](https://github.com/myota-platform/myota-contracts/actions/runs/37814215480),
  and [Helm rendering](https://github.com/myota-platform/myota-deploy/actions/runs/37814665370).
  Fleet deployed Helm revision 88; all 54 tracked resources were Ready, the
  location-enrichment consumer subscribed, and the public gateway health check
  passed. Provider lookup activation and the required pod restart are recorded
  in the follow-up entry above.
- **Geodata service and Admin web:** Added automatically derived Maidenhead
  grid-square and locator arrays to entity resources. Four-character
  (`maidenheadGridSquares4`) and six-character (`maidenheadLocators6`) values
  are calculated from geometry, persisted and backfilled in PostGIS, refreshed
  on geometry changes, and displayed read-only in Entity Catalogue. Points use
  one canonical cell; lines and polygons include each intersected cell. The
  [field reference](../geodata/maidenhead-locators.md) records semantics and
  verification cases. See the [service implementation](https://github.com/myota-platform/myota-geodata-service/commit/3d9a4a4),
  [Admin UI](https://github.com/myota-platform/myota-admin-web/commit/7165c8c),
  [API contract](https://github.com/myota-platform/myota-contracts/commit/a6d7223),
  [platform migration mirror](https://github.com/myota-platform/myota-platform/commit/a2c3556),
  [migration image workflow](https://github.com/myota-platform/myota-platform/commit/d35a5f3),
  and [Helm migration set](https://github.com/myota-platform/myota-deploy/commit/6805e6c).
  The first live migration run exposed a hard-coded runner list ending at 018;
  the runner now discovers every numbered geodata migration in order, and the
  image build watches schema changes. See the [runner fix](https://github.com/myota-platform/myota-platform/commit/8102cd5),
  [service migration guidance](https://github.com/myota-platform/myota-geodata-service/commit/2714dc2),
  [deployment migration guidance](https://github.com/myota-platform/myota-deploy/commit/89b2cfd),
  and [final K3s digest rollout](https://github.com/myota-platform/myota-deploy/commit/051da8f).
  The final Helm migration job completed; live verification confirmed the
  backfill, a boundary-spanning multi-grid geometry, the API fields, and the
  deployed Admin bundle. Exact counts and sample outputs are in the
  [Maidenhead verification record](../geodata/maidenhead-locators.md#live-k3s-verification--8-october-2026).
- **Platform and deployment:** Removed the remaining duplicated migration
  filename lists from both runners. The shared migration runner now discovers
  numbered SQL files in each core, activity, and geodata directory in lexical
  order, and the Compose/deployment copy is synchronized. The database index
  records migration 019 as the current geodata head. This prevents a valid new
  migration from being omitted simply because a separate list was not updated.
  The operator guide and migration ADRs now document the discovery convention,
  synchronization requirement, and that the Helm migration runs as a Job.
  See the [platform runner](https://github.com/myota-platform/myota-platform/commit/90cf589),
  [deployment copy](https://github.com/myota-platform/myota-deploy/commit/1a4708c),
  [operator guide](../operations/production-core.md#migrations-and-recovery),
  [geodata synchronization ADR](../architecture/decisions/0006-geodata-migration-synchronization.md),
  and [three-database ADR](../architecture/decisions/0007-three-database-migration.md).
  The new image was published and its digest recorded in the
  [Fleet rollout commit](https://github.com/myota-platform/myota-deploy/commit/dd68ee2).
  K3s Helm release revision 87 completed the migration Job using that image;
  its logs show numbered migrations discovered through geodata migration 019,
  and Fleet reported 54/54 resources ready.

- **Admin web:** Changed Entity Catalogue page sizes to 25/50/100 and removed
  the overall cutoff from deletion status polling. Bulk jobs are polled in
  bounded batches through transient errors; the modal closes automatically
  when every job reaches a terminal state. A regression test keeps jobs
  polling beyond the previous one-minute limit ([implementation](https://github.com/myota-platform/myota-admin-web/commit/d04ed22)).
  The K3s image was deployed through the [Fleet digest update](https://github.com/myota-platform/myota-deploy/commit/cec0dd3).
- **Admin web:** Fixed bulk permanent deletion's silent no-modal failure. The
  warning dialog now appears before asynchronous per-entity impact lookups;
  lookups are bounded in batches, failures stay visible with a retry option,
  and stable idempotency keys prevent retries from creating duplicate
  confirmation jobs. The final user confirmation remains the only step that
  enqueues deletion events ([implementation](https://github.com/myota-platform/myota-admin-web/commit/c98a507)).
  The updated admin image was published and rolled out to K3s through the
  [Fleet digest update](https://github.com/myota-platform/myota-deploy/commit/d1c1346).
- **Geodata service:** Fixed single and bulk permanent deletion jobs being
  acknowledged without execution when a worker's process-local row cache did
  not yet contain a job created by the API. Workers now reload the job from
  PostgreSQL before claiming it, treat missing jobs as failures, and reconcile
  queued or lease-expired jobs even if a broker event was already acknowledged.
  The worker reconciliation interval is configurable in Compose and Helm.
  K3s verification recovered the 45 queued jobs observed during diagnosis;
  the database had no queued or processing deletion jobs afterward. Two older
  failed legacy jobs remain intentionally excluded and require an administrator
  to create and confirm new jobs.
  See the [worker recovery](https://github.com/myota-platform/myota-geodata-service/commit/e4cac73),
  [platform mirror](https://github.com/myota-platform/myota-platform/commit/58ee6c9),
  [Compose/Helm configuration](https://github.com/myota-platform/myota-deploy/commit/ff9fe3b),
  and [image digest rollout](https://github.com/myota-platform/myota-deploy/commit/b318fbb).
- **Admin web:** Made individual deletion poll the job resource like bulk
  deletion, gave deletion requests finite timeouts, and allowed the modal to
  close while a confirmed job continues in the background. See the [deletion
  workflow guidance](../operations/runbooks.md#permanent-entity-deletion-recovery) and
  [admin implementation](https://github.com/myota-platform/myota-admin-web/commit/e71c9e5).
- **Deploy/observability:** Fixed Grafana's missing SeaweedFS dashboard. The
  live ConfigMap contained the dashboard, but Grafana's `subPath`-mounted file
  remained zero bytes after a ConfigMap update and provisioning repeatedly
  failed with `EOF`. The Helm pod-template checksum now covers all Grafana
  dashboards and provisioning files, so Fleet/Helm changes restart Grafana.
  Added regression coverage and operational troubleshooting guidance
  ([deployment fix](https://github.com/myota-platform/myota-deploy/commit/106324d)).

- **Geodata service:** Added guarded, read-only PostGIS query-plan evidence for
  representative map and catalogue queries, a bounded read-only live baseline,
  and permanent source-tagged Sevilla scale fixtures. The fixture process was
  refined to promote a 5% sample from each fresh import, split evenly between
  Candidate and Approved. The first 2,500-record import had already been fully
  approved before that change and remains a documented lifecycle exception.
  The [promotion-profile change](https://github.com/myota-platform/myota-geodata-service/commit/5c2595d)
  similarly makes the k6 promotion workload use a 40-record import with one
  Candidate and one Approved entity, rejecting the other 95% from staging.
- **Deploy/observability:** Published SeaweedFS S3 metrics and a Grafana object
  storage dashboard, made Collector rollout react to scrape-configuration
  changes, and prevented overlapping PVC mounts during SeaweedFS rollouts.
  Representative changes: [S3 metrics and dashboard](https://github.com/myota-platform/myota-deploy/commit/85ff9a5),
  [configuration-triggered Collector rollout](https://github.com/myota-platform/myota-deploy/commit/02a8574).
- **Identity:** Allowed `GLOBAL_OPERATOR` accounts past the login-attempt
  throttle, addressing the operational-account lockout encountered during
  guarded test setup ([change](https://github.com/myota-platform/myota-identity-service/commit/42e544e)).
- **Admin web:** Fixed Entity Management bulk deletion so focusing an entity
  no longer clears the checkbox batch, confirmation timeouts are reconciled by
  reading the job resource, and the UI tracks each confirmed job to completion
  or failure before reporting success ([change](https://github.com/myota-platform/myota-admin-web/commit/c50ed7a)).
- **Platform integration:** Normalized gateway telemetry labels, added
  PostgreSQL migration tooling to integration images, synchronized geodata
  lookup migrations, and corrected delivery of confirmed entity-deletion jobs
  ([deletion outbox fix](https://github.com/myota-platform/myota-platform/commit/0b42010)).
- **Documentation:** Recorded Phase 0 evidence at the measured catalogue size
  and reorganized the architecture and operational documentation. The resulting
  conclusions and remaining evidence limits are in the
  [Phase 0 evidence record](../geodata/evidence/phase0-production-evidence-2026-10-08.md).

## 7 October 2026 — durable geodata workers, upload recovery, and operations

- **Geodata service:** Made relational rows authoritative across API replicas
  and workers; separated durable preprocessing and promotion processing from
  the HTTP API; added resumable object-storage uploads, bounded processing,
  import cancellation, replay recovery, and load-test cleanup safeguards.
  Representative changes: [relational authority](https://github.com/myota-platform/myota-geodata-service/commit/5dd6192),
  [resumable uploads and workers](https://github.com/myota-platform/myota-geodata-service/commit/7dd9d75),
  [pending-import cancellation](https://github.com/myota-platform/myota-geodata-service/commit/03e77e4).
- **Admin web:** Added resumable import submission and queue summaries,
  preprocessing cancellation, bulk-deletion confirmation, and live JetStream
  status views. Grafana became reachable through an authenticated Admin UI
  proxy rather than a separate public login surface. Representative changes:
  [resumable uploads](https://github.com/myota-platform/myota-admin-web/commit/156738c),
  [import cancellation](https://github.com/myota-platform/myota-admin-web/commit/1ac5b7c),
  [JetStream status](https://github.com/myota-platform/myota-admin-web/commit/e17603f),
  [authenticated Grafana proxy](https://github.com/myota-platform/myota-admin-web/commit/fcef155).
- **Contracts:** Published resource contracts for resumable geodata upload
  sessions, import cancellation, JetStream inspection, and optimistic updates
  ([upload contract](https://github.com/myota-platform/myota-contracts/commit/f430f18),
  [cancellation contract](https://github.com/myota-platform/myota-contracts/commit/4b2ca7f)).
- **Operations service:** Added authenticated, read-only JetStream inspection
  and durable status history, with bounded HTTP concurrency
  ([service implementation](https://github.com/myota-platform/myota-operations-service/commit/047f111)).
- **Activity service:** Changed notification handling to a shared durable pull
  consumer suitable for replicas, and added lifecycle retention for completed
  and failed ADIF source objects ([consumer fix](https://github.com/myota-platform/myota-activity-service/commit/3409dc6),
  [ADIF retention](https://github.com/myota-platform/myota-activity-service/commit/02008e5)).
- **Identity service:** Added least-privilege permissions for operational
  visibility ([change](https://github.com/myota-platform/myota-identity-service/commit/e8478c6)).
- **Deploy and integration:** Wired the operations service and separate
  geodata-worker workloads into Compose/Helm, synchronized schema and runtime
  mirrors, and added graceful notification-consumer draining during rollout.
  Representative changes: [JetStream status and relational geodata rollout](https://github.com/myota-platform/myota-deploy/commit/9f1dace),
  [separately scalable geodata workers](https://github.com/myota-platform/myota-deploy/commit/77c56ec),
  [platform integration synchronization](https://github.com/myota-platform/myota-platform/commit/c080d14).

## 6 October 2026 — guarded workload profiles and production recovery

- **Geodata service:** Added guarded k6 workload profiles, baseline metrics,
  query evidence tooling, and recovery behavior for stale or interrupted
  imports. The harness separated cleanup-only recovery from load generation and
  reported sanitized API failures. Representative changes: [bounded profiles and queue metrics](https://github.com/myota-platform/myota-geodata-service/commit/60da35e),
  [production-safe baseline metrics](https://github.com/myota-platform/myota-geodata-service/commit/a082e69),
  [idempotent import recovery](https://github.com/myota-platform/myota-geodata-service/commit/bc9d092).
- **Contracts and integration:** Specified tagged load-test cleanup and
  synchronized its contract into the platform bootstrap ([contract](https://github.com/myota-platform/myota-contracts/commit/55eadf7)).
- **Deploy:** Connected geodata to the Activity API in Kubernetes and added
  explicit temporary gates for production-targeted load-test cleanup
  ([service routing](https://github.com/myota-platform/myota-deploy/commit/6cbbb5d)).

## 5 October 2026 — retention and service-level telemetry

- **Geodata service:** Added import retention and queue/load-test observability,
  including cleanup of stored source material and run-specific operational
  records after the retention window ([retention](https://github.com/myota-platform/myota-geodata-service/commit/372df64),
  [load profiles](https://github.com/myota-platform/myota-geodata-service/commit/60da35e)).
- **Activity service:** Split award backgrounds, signatures, and issued
  certificates into purpose-specific object-storage buckets and began 15-day
  retention of source ADIF objects ([bucket separation](https://github.com/myota-platform/myota-activity-service/commit/2cec7c6)).
- **Contracts/platform:** Documented current JetStream event behavior and
  synchronized load-test cleanup, award bucket, and import-retention contracts.
- **Admin/deploy:** Put observability behind authenticated Admin UI routing and
  added service and API availability/latency dashboards and alerts.

## 1 October 2026 — resource API migration and three database boundaries

- **Contracts:** Froze and reconciled the canonical REST contract, then
  published typed resource APIs for identity/activity, geodata, and awards;
  recorded route inventory and client migration phases. Representative commits:
  [contract freeze](https://github.com/myota-platform/myota-contracts/commit/2e48230),
  [geodata resources](https://github.com/myota-platform/myota-contracts/commit/45211bd),
  [activity and award resources](https://github.com/myota-platform/myota-contracts/commit/32def3c),
  [typed resource clients](https://github.com/myota-platform/myota-contracts/commit/9a869b1).
- **Geodata service:** Moved PostGIS ownership into `myota_geo` and advanced
  relational persistence for import review ([database ownership](https://github.com/myota-platform/myota-geodata-service/commit/3d1e49a)).
- **Activity service:** Moved normalized activity and award persistence into
  `myota_activity`, added versioned activity/award jobs and metrics, and
  migrated cross-service entity deletion to resource APIs ([dedicated database](https://github.com/myota-platform/myota-activity-service/commit/e8fc7c4),
  [activity/award jobs](https://github.com/myota-platform/myota-activity-service/commit/e538073)).
- **Admin web and participant web:** Migrated writes toward the canonical
  resource APIs, including identity administration, lifecycle changes, and
  participant-facing activity resources ([admin resource writes](https://github.com/myota-platform/myota-admin-web/commit/30c9c21),
  [participant API migration](https://github.com/myota-platform/myota-web/commit/bfca9ee)).
- **Deployment/platform:** Separated storage into three database targets:
  plain PostgreSQL for `myota_core` and `myota_activity`, and PostgreSQL/PostGIS
  for `myota_geo`; synchronized service-owned migrations and deployment
  orchestration ([database-boundary synchronization](https://github.com/myota-platform/myota-platform/commit/5c9b1c2)).
- **All deployable services:** Added GitHub Container Registry publishing
  workflows and aligned image naming with Helm values.

## 30 September 2026 — large file intake and catalogue filtering

- **Admin web:** Added hierarchical continent/country/region/province/city
  filters, removed the redundant status selector, repaired refresh behavior,
  and enabled large resumable file intake through the web proxy. Imports became
  visible while pending preprocessing.
- **Platform/deploy:** Reworked object uploads for streaming and larger request
  capacity while moving away from the original MinIO dependency toward a
  provider-neutral S3 interface and SeaweedFS deployment.
- **Geodata service:** Improved restart recovery for queued imports and made
  import persistence incremental for large datasets.

## 29 September 2026 — Vue administration migration reaches parity

- **Admin web:** Migrated the administration client to Vue 3, TypeScript, and
  Vite; restored dashboard, review, identity, award, programme, and import
  workflows; restored Leaflet and OpenStreetMap rendering, geometry editing,
  clustering, and the map asset pipeline. The legacy implementation was then
  removed after parity work ([Vue administration migration](https://github.com/myota-platform/myota-admin-web/commit/ed9e358),
  [Leaflet parity](https://github.com/myota-platform/myota-admin-web/commit/45256ce),
  [clustering](https://github.com/myota-platform/myota-admin-web/commit/38542f1)).
- **Contracts:** Clarified that preprocessed imports are staged separately and
  only explicit administrator decisions materialize entities.

## 28 September 2026 — staged import validation and review queues

- **Geodata service:** Introduced a durable preprocessed-candidate queue
  separate from Geodata Review. Administrators could validate, reject, or
  promote selected staged records to Candidate or Approved through a processing
  queue; run finalization removed staged records while retaining a summary.
- **Contracts/platform:** Added staged-import, deduplication, finalization, and
  pending-only queue contracts and synchronized the bootstrap implementation.
- **Admin web:** Added paged validation, selection, duplicate comparison maps,
  import summaries, and review-queue actions.

## 27 September 2026 — candidate-only lifecycle and service ownership

- **Geodata service:** Consolidated the entity lifecycle around Candidate,
  Approved, Rejected, and Retired; removed Proposed as a separate entity status.
  Community proposals became a source of Candidate records. Added duplicate
  matching and staged-import API semantics.
- **Contracts:** Captured candidate-only review and staged promotion contracts.
- **Identity, programme, and activity services:** Clarified their service
  boundaries and made durable storage configuration fail fast rather than
  silently falling back to ephemeral state.

## 25–26 September 2026 — shared catalogue, entity metadata, and storage portability

- **Programme service:** Added a shared entity-category master-data catalogue,
  many-to-many programme assignments, and support for multiple allowed
  geometries per category ([catalogue](https://github.com/myota-platform/myota-programme-service/commit/4ac299a),
  [multi-geometry categories](https://github.com/myota-platform/myota-programme-service/commit/25b4b2c)).
- **Geodata service/contracts:** Expanded entity location metadata, added
  hierarchical location options, entity names, multiple category assignments,
  point/line/polygon support, and import/provenance contracts. Manual edits
  were defined to take precedence over automatically fetched location values.
- **Activity service:** Replaced the MinIO-specific client with a provider-neutral
  S3-compatible object-storage interface ([storage portability](https://github.com/myota-platform/myota-activity-service/commit/84b0f7e)).
- **Identity service:** Established the amateur-radio-specific boundary for
  account/person identity, callsign associations, programme-aware claims, and
  administration rather than depending on Keycloak.

## 24 September 2026 — durable domain services and award execution

- **Identity service:** Added durable authentication, bootstrap administrator
  and administration endpoints, and a role catalogue.
- **Programme service:** Added authenticated administration, content/policy
  version workflows, and programme configuration foundations.
- **Activity service:** Added relational persistence in place of the prototype
  JSON state store; created activation/QSO primitives, programme-owned award
  definitions and progress, background workers, notification idempotency, and
  certificate rendering/storage. Awards were exposed through the Activity
  service rather than a separate port ([relational state](https://github.com/myota-platform/myota-activity-service/commit/391366a),
  [award lifecycle](https://github.com/myota-platform/myota-activity-service/commit/9ef3174),
  [activity-owned award execution](https://github.com/myota-platform/myota-activity-service/commit/8a291fd)).
- **Participant web:** Added the interactive public map and participant award
  progress/request flows ([map](https://github.com/myota-platform/myota-web/commit/06b451d),
  [award participation](https://github.com/myota-platform/myota-web/commit/a8cf154)).
- **Administration web:** Established global administration for programmes,
  identity, geodata review, activity, and awards.
- **Contracts/platform/deploy:** Began shared runtime, API/event contract,
  Compose, and Kubernetes deployment integration across the emerging services.

## 23 September 2026 — initial multi-service platform slice

- **Platform and service repositories:** Created the initial Identity,
  Programme, Activity, Geodata, and participant-web implementations from the
  earlier MPOTA application domain, beginning the separation between platform
  capabilities and programme configuration.
- **Identity/programmes/geodata:** Established initial account and callsign,
  programme, entity, and geospatial API foundations.
- **Activity and web:** Added initial activation/QSO and participant UI
  foundations.
- **Architecture principle:** MPOTA was retained only as sample programme
  configuration; programme-specific rules, awards, and eligibility remain
  defined by each programme rather than copied from POTA or another charter.

## Related documentation

- [Current repository and service ownership](../architecture/repository-map.md)
- [Current architecture](../architecture/overview.md)
- [REST API consolidation plan](../domain/api/rest-consolidation-plan.md)
- [Geodata scaling roadmap](../geodata/horizontal-scaling-roadmap.md)
- [Project status and documentation index](../README.md)
