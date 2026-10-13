## 13 October 2026 — Isolated test deployment verified after latest image rollout

- **Prompt used:** “In a new cluster namespace `myota-test`, create a helm
  deployment that will deploy the entire system. Expose the admin ui for that
  deployment as `https://admin.test.myota.top` and the user UI as
  `https://test.myota.top`. Make sure helm picks up new images with the
  testing tag and deploys and reconciles them automatically when a new image
  is published. Also, on first deployment copy data existing in the current
  live system to this second deployment.”
- Added the isolated Helm/Fleet release in Deploy `testing`, installed the
  release into `myota-test`, and seeded its Core, Activity, Geodata, and
  SeaweedFS data from the live namespace. The copy preserved target-owned
  migration/reference records and omitted live delivery/recovery and
  observability state. The first review incorrectly reported the Identity
  source as empty after counting legacy Core `public.account`; Identity stores
  accounts as JSONB in `service_state`, and those records were not copied.
- Deploy PRs #41–#43 made the shared chart upgrade-safe, bumped its package
  version through `0.2.25`, and made stream and durable provisioning run as
  retryable first-install Jobs. The test Helm release was reinstalled from
  Fleet after its prior Helm history became inconsistent. The three database
  PVCs were preserved; the recreated SeaweedFS volume was seeded again from
  the untouched live source.
- Deploy PR [#44](https://github.com/myota-platform/myota-deploy/pull/44)
  passed all required checks and refreshed the platform and gateway pins to
  digests published by testing commit `d03c3d39`. Fleet now tracks commit
  [`d0556c3`](https://github.com/myota-platform/myota-deploy/commit/d0556c376dbaa830d4bc2508a04252dfa496a777),
  and Helm revision 4 is deployed. Fleet reports Ready (76/76 resources), all 27 Deployments and
  StatefulSets are ready, and all running application image IDs match
  the pinned testing digests. Both requested UIs return HTTP 200 with valid
  TLS. JetStream provisioning validated three streams and ten work durables.
- **Identity seed correction:** a read-only recheck after the login report found
  two accounts in live `service_state.state->'items'` for service `identity`,
  including the reported login; `myota-test` has zero persisted identity
  accounts. The test namespace has a separate bootstrap account configured
  for a different email. The original check counted `public.account`, which is
  not this service's source of truth. The production Identity records were not
  copied, and the previous zero-account conclusion was wrong.
- After the user's explicit authorization, copied only the requested active
  account, its password hash, and its active `GLOBAL_OPERATOR` assignment into
  test. Reset its test lockout counters; did not copy other production users,
  sessions, recovery tokens, or security history. Restarted the test Identity
  pod. Verified the password hash matches, account is active, role is present,
  and Identity metrics report the loaded account. No password was available to
  this session, so an actual login was not attempted.
- The `myota-geodata-imports` bucket was independently compared by object key
  and size after the recovery: 10 objects, 13,625,617 bytes. Temporary source
  credentials, copy and verifier Jobs, and ConfigMaps were removed.
- Automatic digest polling is not yet active. The workflow is on Deploy
  `testing`, but GitHub schedules execute workflows from default branch `main`.
  A manual workflow dispatch verified registry digest lookup but failed at PR
  creation because organization policy denies Actions PR creation; repository
  permission changes are rejected and org policy administration is unavailable
  to this session. The divergent branches also make whole-branch promotion
  unsafe. Fleet reconciles committed test values, and PR #44 pinned the latest
  testing images available during this deployment, but future image
  publications still require a deliberate digest update on `testing`. See the
  [test deployment record](docs/operations/test-deployment.md).

## 13 October 2026 — Operations Service repository archived

- **Prompt used:** “Archive the repository `myota-operations-service`.”
- **Pre-archive checks:** The Operations runtime retirement evidence and linked
  Deploy, Contracts, Admin, and Docs work were complete. There were no open PRs;
  issue #3 was the repository's only open issue.
- **Action and verification:** Archived
  [myota-operations-service](https://github.com/myota-platform/myota-operations-service)
  through GitHub. The API then returned `archived: true`; `main` remains its
  default branch. Closed [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3)
  after recording the archive result. No runtime or database changes were made.
- **Record:** [Operations retirement evidence](docs/operations/evidence/operations-service-retirement-2026-10-11.md).

## 13 October 2026 — Data-preserving database rebuilds completed

- **Prompt used:** Work the open database baseline/rebuild issues from
  `.github/profile/README.md` to completion under the standing
  `AGENTS.md`; follow-up: “You haven't updated and checked-off the issues and
  stopped work. Continue.”
- Rebuilt `myota_core`, `myota_activity`, and `myota_geo` from their
  authoritative `main` baselines after quiescing each database's writers,
  relays, workers, and scheduled work.
- The exact post-quiescence archives passed full isolated and baseline
  data-only restores. The immutable runner is deployed; Helm revision 288 and
  Fleet 71/71 are Ready. Identity, Programme, Activity, and Geodata health
  checks returned 200 with durable mode enabled.
- Closed Deploy #16, Activity #5, and Geodata #9 with all acceptance criteria
  checked. Removed the rehearsal namespace, temporary Jobs and client pods,
  test databases, local archives, and copied SQL after recording their
  checksums and restore evidence.
- See the [rebuild evidence](docs/operations/databases/database-rebuild-evidence-2026-10-13.md)
  and [baseline inventory](docs/operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Database baseline gates completed

- Closed baseline issues Deploy #15, Activity #4, and Geodata #8 after source-linked catalog mapping, isolated schema comparisons, and focused API/worker/outbox checks.
- Added source-owner maps for Core's 29 and Geodata's 27 public tables. Disposable Core and Activity API/worker/outbox tests passed; Geodata baseline CI run 38370613580 and quality run 38370614459 passed. Temporary test databases and data were removed.
- Merged Platform mirror PR #5 to main. Opened promotion PRs Deploy #20, Activity #10, and Geodata #12 for the tested migration sources. The exact image digest is not yet deployed to Fleet.
- Rebuild issues Deploy #16, Activity #5, and Geodata #9 remain open. The marked archive/restore evidence predates quiescence and cannot authorize cutover; no live database has been modified.
- See the [baseline and rebuild inventory](docs/operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Activity migration inventory checkoff

- Checked Activity #4's historical migration/object inventory after documenting all eight former migration files, 26 baseline tables, source ownership, and Deploy/Platform mirror paths in the canonical inventory.
- Merged Activity README correction in PR #9 after Python quality run 38376070414 passed.
- Activity #4 remains open for focused API/worker/outbox integration checks. Core and Geodata baseline issues remain open for remaining object ownership and integration evidence. Paired rebuild issues remain gated; no live database was changed.
- See the [baseline/rebuild inventory](docs/operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Database inventory and ownership follow-up

- Expanded the canonical migration inventory with complete baseline table lists and historical migration paths for Core, Activity, and Geodata.
- Linked Identity/Programme common state and outbox paths, Activity repository/worker/consumer paths, and Geodata relational and import-retention paths. Classified Geodata source snapshot tables as active provenance and its service_state as an archive.
- Updated Activity's README to remove stale sequential-migration instructions; PR #9 merged after Python quality passed.
- Baseline issue acceptance checks now stand at Deploy #15 5/7, Activity #4 4/6, and Geodata #8 5/7. All remain open for full object-to-callsite inventories and focused service integration checks. The three rebuild issues remain gated.
- No live database was modified. See the [baseline/rebuild inventory](docs/operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Database baseline ownership inventory

- Expanded the canonical inventory with exact historical migration paths and schema effects for Core, Activity, and Geodata.
- Confirmed the old Operations JetStream and storage snapshot tables are absent from the Core baseline. Kept unresolved compatibility-object ownership visible: Core `activation`, `qso`, and `service_state`; Activity `service_state`; Geodata `service_state` and `source_snapshot_*`.
- All baseline issues remain open until these owner-path and focused service checks are resolved. Rebuild issues remain gated on deployment of the fixed runner and final post-quiescence archives.

## 13 October 2026 — Database baselines and rebuild gate progress

- Re-read `AGENTS.md`, continued Deploy/Core #15/#16, Activity #4/#5, and Geodata #8/#9, and updated every issue with verified checkmarks or current evidence. All six remain open.
- Merged a PostgreSQL 16 psql compatibility fix to the owning and mirror `testing` branches after the runner rehearsal found unsupported pg_dump restriction commands. Deploy #19, Activity #8, and Platform #4 all passed their CI checks.
- Ran the fixed unified migration runner against fresh disposable Core, Activity, and PostGIS databases. Fresh application and repeat behavior passed. A missing Core sentinel caused the run to fail with readiness false for all targets; restoring the sentinel/checksum allowed recovery. Runner-test databases, jobs, ConfigMap, and temporary source files were removed.
- Preliminary custom-format archives passed inspection and isolated restore; their checksums and limits are in the [baseline/rebuild inventory](docs/operations/databases/baseline-migration-inventory.md). They were captured before quiescence and are not safe cutover backups.
- Helm 282 is a rollback to 281; Fleet still tracks Deploy `main` commit `9d80741`, and the completed live migration job used the historical chain. All live databases remain intact; no rebuild go/no-go was recorded.

## 12 October 2026 — Structured logging and request-to-worker correlation

- Closed Programme #4, Activity #3, and Geodata #6 after deployed producer/worker
  correlation evidence, ACK checks, and exact probe cleanup. Identity #3 was
  already closed.
- Fixed the deployed Activity consumer's NATS header mapping access and enabled
  production OTLP identity for Activity/Geodata workers and outbox relays.
- Audited Loki's indexed labels (three fixed labels), sampled metadata for
  payload leakage, confirmed live 14-day retention configuration and compactor
  runs, and linked 33 Prometheus plus two Grafana alerts to the operator
  runbook.
- Deploy #6 remains open for authenticated Grafana navigation, full Compose
  runtime parity, representative capacity/recovery, and Fleet application of
  the corrected chart. A malformed alert annotation caused Prometheus to restart;
  Helm rollback revision 279 restored it to 1/1. The corrected Prometheus and Grafana rule source passes focused YAML parsing;
  Deploy Python quality passes, while Fleet remains `WaitApplied` with applied
  sync generation 92 versus desired 94. Details:
  [12 October evidence](docs/observability/evidence/logging-implementation-2026-10-12.md)
  and [issue #6](https://github.com/myota-platform/myota-deploy/issues/6).

## 12 October 2026 — Observability dashboard issue recheck

- Rechecked closed Deploy issues [#11](https://github.com/myota-platform/myota-deploy/issues/11),
  [#12](https://github.com/myota-platform/myota-deploy/issues/12), and
  [#13](https://github.com/myota-platform/myota-deploy/issues/13). The active
  Helm 0.2.20 revision 275 is Fleet Ready at Deploy commit c7b6956, with 71/71
  resources and all 23 Deployments/27 replicas ready. K3s workload, application
  workflow, and PostgreSQL/PostGIS dashboards remain provisioned; no test data
  was created. Original query results and known metric gaps remain documented
  in the [evidence record](docs/observability/evidence/dashboard-enhancements-2026-10-12.md).

## 12 October 2026 — Admin entity details save follow-up

- Revisited [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2)
  after the report that Website and Description were not saved by **Save Name**.
  Moved both fields below Entity name and Entity categories and replaced the
  separate saves with one **Save details** action.
- The action uses the existing versioned metadata PATCH and category PUT,
  refreshing the entity version between calls. If the second call fails after
  metadata has saved, the UI reports partial completion and retains the category
  draft for retry.
- Admin commit [d833f5d](https://github.com/myota-platform/myota-admin-web/commit/d833f5dc26ca351684222ec2971e3ea6e4470d4c)
  and test commit [daecbdd](https://github.com/myota-platform/myota-admin-web/commit/daecbddb6537a71517bfc76049701fbd2b346d1e)
  passed Admin workflow [#38187013011](https://github.com/myota-platform/myota-admin-web/actions/runs/38187013011):
  29 unit tests, typecheck, production build, six browser tests, and image publish.
- Deploy pins image `sha256:23f4466f743d7f41d4a3549510926effe71e797aacc1a67c22aff85a6cafec54`
  in [commit 3b6acef](https://github.com/myota-platform/myota-deploy/commit/3b6acef8e45e83e40f6283accfaf52c40902bdb8);
  Platform mirrors it in [commit c51a95e](https://github.com/myota-platform/myota-platform/commit/c51a95ed80a9dd7b702e0b6a3212572b5266bbef).
  K3s serves the new bundle, Admin is Ready 1/1, and all 23 Deployments are ready.
  Helm revision 274 is deployed; Fleet reports 71/71 resources ready. The current
  release state is recorded in [current system state](docs/architecture/current-state.md).
- The [editor guide](docs/domain/administration/entity-catalogue-editor.md)
  and [diagram](docs/architecture/diagrams/entity-catalogue-editor.md) describe
  the final behavior.

## 11 October 2026 — Admin entity metadata visibility follow-up

- Revisited [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2)
  after the report that Website and Description were not visible in the entity
  management pop-up. The fields existed but were nested in the Name & categories
  section; the fix moves them to a dedicated section near the top of the dialog.
- Added browser assertions for the section, both inputs, and loaded values.
  Admin CI run [#38183593814](https://github.com/myota-platform/myota-admin-web/actions/runs/38183593814)
  passed its production build, type check, publish step, and 24 browser tests.
- Deployed image
  `ghcr.io/myota-platform/myota-admin-web@sha256:ac82ae5420a5a449a2c5a812d972e62f3f8a291edbae3e7a3f45695388abdb24`
  from Admin source commit `c4314ba2eef69877219861e24c1220f89801ae3b`.
  Helm revision 273 is `deployed`; Fleet reports Ready and all 23 Deployments
  (27 replicas) are Ready. A read-only live bundle inspection confirmed the
  new management section and fields are served. No test data was created.
- Deployment pins were committed in Deploy
  `0dd7de270a111adf2169601c1e8e796ef203fc57` and synchronized to Platform
  `9cabc46d657e387b5adb2152c294af206dd0342f`. See the [editor guide](docs/domain/administration/entity-catalogue-editor.md),
  [diagram](docs/architecture/diagrams/entity-catalogue-editor.md), and
  [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2).

## 11 October 2026 — Operations Service retirement and OpenTelemetry migration

- Retired the Operations workload, API route/client, image publishing, and
  snapshot tables. Direct least-privilege Collector receivers own PostgreSQL
  metrics; Gateway and Identity own Grafana authorization.
- Normalized monitor credentials and synchronized PostgreSQL roles. Live
  metrics were verified for all databases; unauthenticated Grafana auth
  returns 401 and `Cache-Control: no-store`.
- Helm 0.2.20 revision 272 is deployed and Fleet Ready with 23 Deployments
  and 27 replicas. Operations snapshot tables are absent.
- At the time of this 11 October entry, repository archival was still pending;
  it was completed on 13 October as recorded in the later archive entry below.

## 11 October 2026 — Observability dashboard enhancements

- Added a Helm-only K3s workload dashboard using namespace-scoped
  kube-state-metrics and internal, TLS-verified Kubelet resource/volume
  metrics. Infra-owned RBAC grants only the node metrics reads required by
  Prometheus.
- Added a unified application workflow dashboard and expanded PostgreSQL/
  PostGIS health panels. Unsupported per-durable JetStream delivery signals
  and unavailable database maintenance metrics are called out instead of
  rendered as zero.
- Synchronized Deploy-owned dashboard sources to Platform, deployed chart
  0.2.20 at Helm revision 265, and verified live metrics on K3s. See the
  [evidence record](docs/observability/evidence/dashboard-enhancements-2026-10-12.md)
  and closed Deploy issues [#11](https://github.com/myota-platform/myota-deploy/issues/11),
  [#12](https://github.com/myota-platform/myota-deploy/issues/12), and
  [#13](https://github.com/myota-platform/myota-deploy/issues/13).

## 11 October 2026 — Identity and Programme metrics coverage

- Added bounded HTTP request counters and duration histograms to Identity and
  Programme, with route-template and status labels that exclude account,
  programme, callsign, URL, and other high-cardinality identifiers.
- Added separate Identity and Programme scrape jobs in Deploy and synchronized
  the Platform mirror. CI, metric tests, immutable image builds, Helm chart
  validation, and mirror checks passed.
- Deployed both images in Helm revision 261. Read-only endpoint checks and
  Prometheus queries found non-empty request series under distinct service
  labels. See the [implementation and live evidence](docs/observability/evidence/identity-programme-metrics-2026-10-11.md)
  and the closed issues for [Identity](https://github.com/myota-platform/myota-identity-service/issues/4),
  [Programme](https://github.com/myota-platform/myota-programme-service/issues/3),
  and [Deploy](https://github.com/myota-platform/myota-deploy/issues/7).

## 11 October 2026 — Structured logging and request-to-worker correlation

- Fixed the deployed Activity notification consumer crash caused by calling
  `message.header(...)` instead of reading nats-py's dictionary-backed
  `Msg.headers`. Added header mapping
  and end-to-end correlation regression coverage; synchronized Deploy and
  Platform mirrors.
- Deployed Activity image
  `sha256:33e7b42f68544b569d621f93bae8fdeec5f696408f0527bc484be6a648673bbc`
  in Helm revision 259. Fleet is ready with all 23 Deployments and 27 replicas.
- Verified a sanitized Identity registration through the outbox relay, JetStream,
  and Activity notification worker. The original correlation ID appeared in
  worker lifecycle logs and the durable ACK floor advanced with no pending
  messages.
- Removed the synthetic account/event, outbox record, JetStream message,
  Activity notification, processed marker, and checkpoint after the probe.
  Identity issue #3 is closed; Programme #4, Activity #3, Geodata #6, and Deploy
  #6 remain open for the evidence gates recorded in the logging roadmap.

## 11 October 2026 — Geodata Website and Description fields

- Added nullable entity Website and Description columns with additive migration
  `022_entity_website_description.sql`; legacy entities remain valid and blank.
- Extended the existing versioned metadata API and the OpenAPI contract. Website
  accepts absolute HTTP/HTTPS URLs up to 2,048 characters; Description accepts
  plain text up to 2,000 characters. Omission preserves values; null/blank clears.
- Manual proposals can provide both fields. Import refreshes preserve curated
  values on existing entities. Updates pass the existing review authorization,
  revision check, audit history and transactional outbox, with additive fact
  `geodata.entity.metadata-updated.v1`.
- Admin catalogue and detail views display the values; authorized managers can
  edit or clear them. External links are restricted to HTTP/HTTPS and use
  `rel="noopener noreferrer"`; descriptions render as escaped text.
- Detailed behavior and evidence: [entity catalogue editor guide](docs/domain/administration/entity-catalogue-editor.md),
  [Geodata #3](https://github.com/myota-platform/myota-geodata-service/issues/3),
  [Contracts #3](https://github.com/myota-platform/myota-contracts/issues/3),
  [Admin #2](https://github.com/myota-platform/myota-admin-web/issues/2).

## 11 October 2026 — Geodata TracerProvider warning issues

- Confirmed two process-global tracing provider registrations: startup
  `configure_logging()` installs the provider through `configure_tracing()`,
  then the first HTTP request previously created and registered another
  provider in `otel.telemetry_for()`. Repeated access under another service
  name could repeat that attempt.
- Geodata now uses one process-wide telemetry object and reuses the existing
  tracing API provider. Setup failures are logged with bounded diagnostics
  and leave HTTP handling available. Deploy and Platform runtime mirrors and
  all copies of the shared logger are synchronized; immutable images were
  built and pinned for Fleet.
- The startup/request regression checks, repository workflows, host readiness,
  and a non-mutating `/healthz` trace are recorded in the
  [TracerProvider repair evidence](docs/observability/evidence/geodata-tracerprovider-2026-10-11.md).
  See [Geodata #7](https://github.com/myota-platform/myota-geodata-service/issues/7)
  and [Deploy #14](https://github.com/myota-platform/myota-deploy/issues/14).

## 11 October 2026 — Observability dashboard issues

- Created linked Deploy issues for K3s workload health/capacity, application
  workflow and JetStream delivery, and PostgreSQL/PostGIS health panels. The
  tasks build on existing dashboards, keep cluster metrics private, avoid
  event/query payload labels, and distinguish missing measurements from zero.
- Existing Identity/Programme dashboard coverage and Operations-to-OpenTelemetry
  migration remain tracked separately. See [Deploy #11](https://github.com/myota-platform/myota-deploy/issues/11),
  [#12](https://github.com/myota-platform/myota-deploy/issues/12), and
  [#13](https://github.com/myota-platform/myota-deploy/issues/13).

## 11 October 2026 — Service metrics scrape repair

- Restored bounded aggregate Prometheus endpoints for Identity and Programme,
  aligned Helm's Activity notification scrape target with Compose, and
  synchronized the Deploy-owned changes to Platform.
- Verified all 12 configured scrape endpoints returned HTTP 200 with metric
  samples on host K3s; the Collector's latest log check showed no failed
  scrapes. See [Deploy issue #10](https://github.com/myota-platform/myota-deploy/issues/10)
  for commits, CI runs, deployed digests, and verification details.

# MyOTA changes

## 11 October 2026 — Geodata Phase 5 owner-accepted closure

- Closed Phase 5 administratively on the project owner's instruction, accepting
  the remaining qualification risk. No unverified gate is represented as a
  tested pass; checklist items remain unchecked.
- Broad-catalogue/write, node/storage, canary, rollback, and above-three-replica
  evidence remains unverified and must be reopened before capacity expansion.
- See the [closure record](docs/geodata/evidence/phase5-owner-acceptance-2026-10-11.md)
  and [scaling roadmap](docs/geodata/horizontal-scaling-roadmap.md).

## 11 October 2026 — Geodata Phase 5 scale qualification follow-up

- The read-only 2/10/50-VU catalogue profiles passed with 0% request failures
  and 100% checks; the 50-VU profile measured 95.07 requests/s and 22.94 ms
  p95 on ten entities. No test data or runtime changes were made.
- Added and locally verified a bounded independent-row edit profile for future
  qualification. The write, broad-catalogue, canary, rollback, and
  node/storage gates remain open.
- See the [dated evidence](docs/geodata/evidence/phase5-read-only-follow-up-2026-10-11.md)
  and [Geodata scaling roadmap](docs/geodata/horizontal-scaling-roadmap.md).


## 11 October 2026 — Current system documentation and diagrams

- Added a single current-state snapshot from GitHub main heads and read-only
  K3s/Fleet/Helm evidence. It records release and image digests, ownership, NATS
  streams, completed migrations, open work, and evidence limits.
- Corrected current service, data, NATS, and observability diagrams. Operations
  is no longer shown as a JetStream observer; NATS Surveyor owns broker
  monitoring. Removed stale Interest-retention and old Helm/Fleet status claims
  from current-status sections while preserving historical snapshots.
- Simplified the root README and documentation indexes. Roadmaps retain task
  checklists, dated evidence remains historical, and compatibility pages remain
  available for existing links.
- Documentation-only change; runtime and cluster were not modified. See
  [Current system state](docs/architecture/current-state.md) and the
  [implementation timeline](docs/history/implementation-timeline.md).

# MyOTA changes

## 11 October 2026 — Cross-repository architecture review and mitigation

- Reviewed the service owners, Contracts, Deploy, Platform, both web clients,
  Docs, and the shared GitHub workflow on main. Phases 1–4 then restored
  owner-to-mirror parity, established a synchronized logger source, grouped
  Geodata route registration, and removed unused pre-live API aliases and
  their deprecation machinery.
- Contracts CI passes for 44 owner files, 92 file mirrors, and five exact tree
  mirrors. The current route inventory contains 131 OpenAPI operations and
  170 route registrations with no missing or unregistered routes or duplicate
  operation IDs.
- Fleet is Ready=True at Deploy commit f7d2e65; Helm release myota is
  deployed at revision 246 and all Deployments were ready at requested counts.
- Phase 0 is complete: the duplicate cleanup path was merged into one
  handler-aligned contract; Contracts CI rejects duplicate YAML mapping keys;
  the route inventory has 131 unique operations and 170 route registrations.
  Contracts CI run [38157218999](https://github.com/myota-platform/myota-contracts/actions/runs/38157218999)
  and Platform CI run [38157179490](https://github.com/myota-platform/myota-platform/actions/runs/38157179490)
  passed. The canonical, Contracts root, and Platform copies are byte-identical.
  See the [Phase 0 closure record](docs/architecture/evidence/review-phase-0-2026-10-11.md).


## 11 October 2026 — Phase 7 database observability work

- Added dedicated non-superuser PostgreSQL monitor-role provisioning and
  external Secret references to Deploy Compose and Helm; mirrored the deployment
  assets in Platform.
- Added a six-panel PostgreSQL/PostGIS Grafana dashboard to Compose and Helm.
  The dashboard JSON is mounted and parses in the Grafana pod.
- A read-only smoke queried `pg_stat_database` in core, Activity, and Geo. The
  pinned Collector receiver still failed authentication with the same Secret
  references in both the live and temporary diagnostic Collector. The receiver
  was removed from the active metrics pipeline to stop repeated failures.
- All Deployments report Ready. Helm revision 234 is deployed; Helm history labels it a rollback to revision
233. The Fleet BundleDeployment is Ready=True at sync generation 93. All 67
resources are ready, and all Deployments meet their requested replica counts. Phase 7 remains open; no PostgreSQL receiver metrics are
  claimed as delivered. See the [evidence](docs/observability/evidence/logging-implementation-2026-10-11.md)
  and [plan](docs/observability/logging-implementation.md).

## 11 October 2026 — Structured logging rollout and live trace verification

- CI-published logging images for Identity, Programme, Activity, and Geodata
  are running on K3s; all service and worker replicas were Ready at verification.
  Geodata relational-boundary, load-harness, and upload-recovery jobs passed.
- A read-only Identity `GET /healthz` returned HTTP 200. Its trace ID matched
  one Loki `http.request.completed` structured-metadata record and one Tempo
  `GET /healthz` span. No business data was written.
- Live Loki confirmed only three indexed label names. OTLP attributes are
  queryable as structured metadata, so the JSON parser was removed from the
  Grafana dashboard queries and runbook; the trace link now selects the
  `trace_id` metadata field.
- Deploy and its Platform mirror contain the dashboard/datasource correction.
  Helm revision 225 mounted the corrected datasource/dashboard, Grafana logged successful provisioning, and all seven mounted dashboard queries returned successfully against live Loki. Fleet is Ready=True on Deploy commit `904bef2`; authenticated UI navigation remains open. Helm revision 227 is deployed after rollback to 226.
  Retention growth, safe live redaction, representative cardinality, async
  request-to-worker correlation, and PostgreSQL/PostGIS observability remain
  open. See the [plan](docs/observability/logging-implementation.md),
  [evidence](docs/observability/evidence/logging-implementation-2026-10-11.md),
  and [runbook](docs/observability/logging-runbook.md).

## 11 October 2026 — NATS migration Phase 6 fact-stream cutover

- Completed the direct, fail-closed `MYOTA_EVENTS` migration from Interest to
  bounded Limits retention after verifying the stream and Activity durable were
  empty/drained and all three owner outboxes had no unpublished rows. Removed
  the legacy `myota.geodata.>` capture; the six Activity and four Geodata work
  durables remain on their separate WorkQueue streams.
- Set finite one-node caps: Events 1 GiB/500,000/1 MiB, Activity work
  1 GiB/250,000/1 MiB, and Geodata work 3 GiB/500,000/1 MiB. Each stream is
  file-backed, one replica, 30-day max age, and DiscardNew. The one-time
  migration flag is disabled; topology validation remains enabled.
- Helm revision 213 is deployed and MyOTA Fleet is Ready=True on Deploy main
  commit `dfc423d26bd79a82b1177b9e5d51e2d1196580c9`; all MyOTA Deployment
  replicas are ready. Deploy Python quality and Helm validation passed
  (runs 38099669717 and 38099669405).
- A no-message disposable K3s migration/idempotency check passed and its
  namespace was removed. No production test records were created. Preserved
  all six existing Geodata database DLQ records; previously Interest-expired
  facts remain unavailable. PostgreSQL remains authoritative, and off-node
  broker recovery remains deferred. No Phase 6 schema purge was applicable.
  No metrics baseline, shadow traffic, or canary was added.
- See the [Phase 6 evidence](docs/operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md),
  [migration plan](docs/operations/messaging/nats-event-migration-plan.md),
  [ADR-0008](docs/architecture/decisions/0008-nats-jetstream-event-and-work-topology.md),
  and [topology diagram](docs/architecture/diagrams/nats-event-migration.md).


## 11 October 2026 — NATS Surveyor monitoring implementation

- Surveyor 0.9.11 and all eleven pinned Grafana dashboards are live. After a
  temporary exporter restart, Prometheus reported only the active NATS server
  ID, `up=1`, `nats_up=1`, and the configured 7 GiB file-store capacity. The
  1 GiB memory cap and 15-day Prometheus retention are active.
- The Admin NATS page, Operations broker sampler/API, and Geodata duplicate
  poller are retired in their owning images. Operations migration 003 is
  mirrored as Deploy/Platform core migration 005; Helm revision 208 applied it.
  The refreshed Operations runtime image is pinned at
  `sha256:b50b4666919e1f4046067129abd95993d371093088ea5e5f7633fd5ced163af4`.
  The snapshot table/index are absent; the storage history table and verified
  external backup remain.
- A memory-only validation stream received one synthetic message and appeared
  in Surveyor metrics. The stream and temporary pod were removed. Grafana lists
  eleven dashboards in folder `NATS`; the old JetStream backlog and PostGIS
  dashboards are absent, and their two PostGIS panels were not recreated.
- After restarting the local Fleet agent and issuing one force sync, Fleet
  reports Ready=True at Deploy main commit
  `d0d2f50846fe2c9e239d46a48809860f4b98d7f5`, with 64/64 resources ready and
  applied/desired IDs matching. Phase 2 is complete. See the
  [monitoring roadmap](docs/observability/nats-surveyor-migration.md) and
  [evidence record](docs/observability/evidence/2026-10-11-nats-surveyor-cutover.md).
- Follow-up live query validation executed all thirteen dashboard 16256
  expressions through Grafana's Prometheus datasource API; every expression
  returned HTTP 200 and a data frame with no query error. The NATS folder
  contains eleven dashboards and no NATS alerts are firing. Five unrelated
  application alerts were active at the same snapshot; they are recorded for
  separate follow-up in the evidence.

## 11 October 2026 — Phase 5 observation closure and legacy durable retirement

- Helm revision 193 remained deployed. Fleet was Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a; all MyOTA Deployments were
  Ready. Fleet's observation anchor was 22:03:29 UTC on 10 October.
- At the user's explicit direction, the 24-hour elapsed-time requirement was
  waived. Final checks and retirement occurred at approximately 22:13:48 UTC
  on 10 October, roughly 10 minutes 19 seconds after the anchor. This is an
  authorized early close, not a claim that 24 hours elapsed.
- Before retirement, both file-backed streams were empty. The four legacy
  filters and four replacement filters matched expectations; all counters
  were zero, target workers had one waiting pull, and Activity notification
  remained healthy. Migration 021 columns and indexes were present, and
  age-bounded recovery checks found zero due owner rows.
- Deleted and individually verified absent only these MYOTA_EVENTS consumers:
  geodata-entity-deletion-v1, geodata-import-processing-v2,
  geodata-location-enrichment-v1, and geodata-preprocessing-v1. Post-check
  confirmed MYOTA_EVENTS and its Activity durable remain, along with
  MYOTA_GEODATA_WORK and its four target durables.
- An isolated in-memory broker deletion-semantics check passed without
  publishing messages. The retirement-test namespace and the earlier
  recovery/delivery validation namespace were deleted and verified absent.
  No production test data, messages, rows, schema objects or recovery records
  were changed.
- Two full delivery-suite attempts each had one timing-sensitive immediate
  ACK-counter assertion. Focused ACK verification passed and later isolated
  broker metadata settled at zero; a clean full-suite pass is not claimed.
  See the [Phase 5 evidence](docs/operations/messaging/evidence/phase5-geodata-work-2026-10-10.md)
  and [migration plan](docs/operations/messaging/nats-event-migration-plan.md).


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

## 11 October 2026 — Logging database telemetry and current rollout

- Replaced the failed direct Collector PostgreSQL receiver with the Operations
  service's read-only bounded monitor adapter for core, Activity, and Geo.
- Enabled pinned Psycopg instrumentation across service images, with parameter
  capture and SQL commenter disabled; Collector removes SQL attributes before
  Tempo export.
- Live samples show all three databases up, bounded connection/size/counter
  metrics, and PostGIS 3.4.3 present in Geo. A read-only trace test produced
  three SQL-free child spans. No business rows were written.
- Added Loki internal-metric scraping and capacity dashboard panels. The short
  initial bucket sample is explicitly not treated as a production growth rate.
- Deploy main commit
  [86df3de](https://github.com/myota-platform/myota-deploy/commit/86df3dedfeffa42fe44011169bec0e81eae36713)
  is deployed at Helm revision 241; Fleet reports 67/67 resources ready.
  Remaining logging acceptance: authenticated Grafana navigation, async
  correlation, Loki field privacy audit, and Compose runtime parity. See
  [current evidence](docs/observability/evidence/logging-implementation-2026-10-11.md)
  and the [plan](docs/observability/logging-implementation.md).
## 11 October 2026 — Stable NATS correlation context

- Outbox relays now propagate a stable message correlation header using the
  envelope correlation ID, work ID, or event ID. Consumer log context inherits
  the header; focused tests and service/deployment CI passed.
- The five service images and shared Deploy workers were rebuilt and rolled
  out at the digest values recorded in the
  [logging evidence](docs/observability/evidence/logging-implementation-2026-10-11.md).
- All Deployments are Ready. Helm revision 242 still reports `pending-upgrade`
  while Fleet reports revision 241 Ready with 67/67 resources. Request-to-worker
  correlation remains open where the source outbox does not store the HTTP ID.

- **Rollout note:** all Deployments are ready on the updated image digests, but Helm revision 242 remains `pending-upgrade`; Fleet still reports revision 241 Ready with 67/67 resources. The release-state discrepancy remains open.

## 11 October 2026 — SeaweedFS monitoring retirement

- Replaced the Admin monitoring page with a legacy-bookmark redirect to the
  provisioned Grafana dashboard adapted from Grafana Labs dashboard 19173.
- Removed Operations SeaweedFS sampling, API routes, settings and sampler module;
  retained read-only PostgreSQL/PostGIS metrics, Grafana role resolution and
  application storage/readiness behavior. Added an idempotent forward migration
  that drops only `operations_storage_snapshot` and its owned index.
- Removed the two Operations monitoring paths/schema from OpenAPI and generated
  clients. Updated Deploy/Platform Compose, Helm, Operations mirrors, dashboard
  and alerts, architecture diagrams, runbooks, READMEs and repository map.
- Verified Admin unit tests (28), browser tests (23), typecheck and production
  build; Operations Ruff and six unit tests; Contracts seven unit tests, two
  generated clients, route inventory and Deploy route audit; Helm lint and
  rendered chart validation; Compose and chart YAML parsing. Deploy and Admin
  image workflows passed.
- On local K3s, the revision 249 migration completed and checks confirm the
  sampled table/index and retired endpoints are absent, `/healthz` returns 200,
  and all 14 dashboard Prometheus targets return live series. All 23 Deployments
  and 67 Fleet resources report ready. Helm revision 249 is deployed; Fleet is
  Ready 1/1 on Deploy commit `12251ff0ccea4c01ac039cc85cd1ee44e29d1b95`, with
  matching applied and desired deployment IDs. The initial pending-upgrade state
  cleared during Fleet reconciliation.
- Commits: Operations `1b52897`, Deploy `6a87f36`, Platform `eb79df0`, Contracts
  `3852666`/`cce166a`, and Admin `9d54cc6`; all pushed to `main`. See the
  [current state](docs/architecture/current-state.md), [monitoring guide](docs/operations/storage/seaweedfs-admin-status.md),
  and [cross-repository issues](https://github.com/myota-platform/myota-deploy/issues/8).


## 11 October 2026 — NATS migration evidence and readiness closeout

- Closed the Contracts, Identity, Programme, Activity, Geodata, and Deploy
  NATS migration evidence issues. Closed Operations #1 as superseded by the
  approved Surveyor monitoring and Operations retirement work.
- Updated the contract registry check to 68 facts and 10 work commands and
  synchronized its Platform mirror. A clean-clone registry test passed 7/7;
  the workspace event-source audit found no undispositioned event-like source
  literals.
- Read-only K3s inspection confirmed the three bounded streams, 11 expected
  consumers, zero current backlog, three ready outbox relays, Helm revision
  257, Fleet Ready 1/1, and 23/23 ready Deployments (27/27 replicas). The
  temporary inspection pod was deleted and published no messages.
- Off-node recovery remains deferred; the accepted single-node capacity
  ceilings are not a representative 30-day capacity qualification. Six
  unresolved Geodata application DLQ records remain preserved. The current
  Contracts workflow mirror-audit and Platform Ruff caveats are documented.
  See the [NATS readiness closeout](docs/operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md),
  the [migration plan](docs/operations/messaging/nats-event-migration-plan.md),
  and the current [system snapshot](docs/architecture/current-state.md).
