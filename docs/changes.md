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
The [repository map](repository-map.md) and [architecture](architecture.md)
describe current ownership and are authoritative for the present-day system.

## 8 October 2026 — scale evidence and deployment observability

- **Geodata service and Admin web:** Added automatically derived Maidenhead
  grid-square and locator arrays to entity resources. Four-character
  (`maidenheadGridSquares4`) and six-character (`maidenheadLocators6`) values
  are calculated from geometry, persisted and backfilled in PostGIS, refreshed
  on geometry changes, and displayed read-only in Entity Catalogue. Points use
  one canonical cell; lines and polygons include each intersected cell. The
  [field reference](geodata-maidenhead-locators.md) records semantics and
  verification cases. See the [service implementation](https://github.com/myota-platform/myota-geodata-service/commit/3d9a4a4),
  [Admin UI](https://github.com/myota-platform/myota-admin-web/commit/7165c8c),
  [API contract](https://github.com/myota-platform/myota-contracts/commit/a6d7223),
  [platform migration mirror](https://github.com/myota-platform/myota-platform/commit/a2c3556),
  and [Helm migration set](https://github.com/myota-platform/myota-deploy/commit/6805e6c).

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
  workflow guidance](operations.md#permanent-entity-deletion-recovery) and
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
  [Phase 0 evidence record](geodata-phase0-production-evidence-2026-10-08.md).

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

- [Current repository and service ownership](repository-map.md)
- [Current architecture](architecture.md)
- [REST API consolidation plan](api-rest-consolidation-plan.md)
- [Geodata scaling roadmap](geodata-horizontal-scaling-roadmap.md)
- [Project status and documentation index](README.md)
