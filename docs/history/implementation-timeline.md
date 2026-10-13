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

## 13 October 2026 — Core, Activity, and Geodata rebuild closeout

- **Prompts used:** “Work on the issues under ‘Database migration baselines
  and data-preserving rebuilds’ to completion following the standing
  instructions in `./AGENTS.md).” Follow-up: “You haven't updated and
  checked-off the issues and stopped work. Continue.”
- **Scope:** Finish Deploy/Core #16, Activity #5, and Geodata #9 in the local
  non-production K3s cluster; verify exact backups/restores and service recovery;
  update the database evidence and issue index.
- **Evidence:** Final quiescent Core (106,726 bytes), Activity (864,682 bytes),
  and Geodata (9,649,351 bytes) archives passed `pg_restore --list`, full
  isolated restore, and baseline data-only restore. Baseline Git blob SHAs,
  PostgreSQL/PostGIS versions, archive checksums, catalog counts, retained rows,
  grants, queue/durable state, API health, cleanup, and recovery steps are in
  [the closeout record](../operations/databases/database-rebuild-evidence-2026-10-13.md).
- **Runtime:** The unified runner image completed Helm migration Job
  `myota-migrations-288`. Helm is deployed at revision 288; Fleet is Ready
  71/71. Identity, Programme, Activity, and Geodata health endpoints returned
  200 with durable mode enabled. All three database readiness markers are
  true at revision 288.
- **Issue completion:** Deploy #16, Activity #5, and Geodata #9 are each 7/7
  checked and closed. The three baseline issues remain closed. The rehearsal
  namespace, helper Jobs/pods, temporary databases, archives, and copied SQL
  were removed after evidence capture.
- **Record:** [Database baseline inventory](../operations/databases/baseline-migration-inventory.md)
  and [rebuild closeout evidence](../operations/databases/database-rebuild-evidence-2026-10-13.md).

## 13 October 2026 — Database baseline acceptance and promotion

- **Prompt used:** “Work on the issues under ‘Database migration baselines and data-preserving rebuilds’ in the organization issue index to completion following the standing instructions in `./AGENTS.md).” Follow-up: “You haven't updated and checked-off the issues and stopped work. Continue.”
- **Evidence:** Core and Activity fresh baseline catalogs were compared with the archived schemas; each contained the expected new `myota_schema_migration` ledger, while application objects and deployment readiness remained. Identity create/read persistence and outbox, Programme create/read persistence and outbox, and Activity create/read plus worker lease/complete/outbox paths passed against disposable baseline databases. Geodata CI run 38370613580 passed baseline migration, two API instances, worker ACK/recovery, and upload recovery; PostGIS and spatial-index evidence passed. Temporary test DBs/data and copied SQL were removed.
- **Issue completion:** Deploy #15 (7/7), Activity #4 (6/6), and Geodata #8 (7/7) are closed. The rebuild issues #16/#5/#9 remain open at 1/7 each; only preliminary pre-quiescence archive/restore evidence is checked.
- **Promotion:** Platform PR #5 merged to main at `a421a3a281d2653de145f8bb38e40e1539257fb4`. Deploy #20, Activity #10, and Geodata #12 are awaiting quality gates/promotion. The image digest and Fleet deployment have not yet changed, so no live migration run or database cutover was performed.
- **Record:** [Database baseline and rebuild inventory](../operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Activity inventory acceptance check

- **Prompt used:** Continue the database baseline and data-preserving rebuild issues under the standing `AGENTS.md` instructions.
- **Evidence:** Activity's historical migrations 001–008 are mapped to schema effects; all 26 baseline tables and source/mirror paths are listed. Activity README PR #9 replaced stale sequential migration instructions and linked the canonical inventory.
- **Issue update:** Activity #4's migration inventory acceptance criterion is now checked. Its focused API/worker/outbox disposable-database checks remain open; issue #4 remains open. Deploy #15 and Geodata #8 also remain open with their unverified object mappings and integration gates. Paired rebuild issues #16/#5/#9 remain gated.
- **Runtime boundary:** No live database was changed. The fixed runner has only been tested in an isolated namespace and is not yet deployed in Helm.

## 13 October 2026 — Database object mapping and source guide update

- **Prompt used:** Continue work on the database baseline and data-preserving rebuild issues under the standing `AGENTS.md` instructions.
- **Scope:** Refine the historical migration/object inventory and update source guides and issue evidence. No runtime or live database changes.
- **Evidence:** The inventory now lists all Core, Activity, and Geodata baseline tables and exact historical migration paths. Source inspection linked Identity/Programme state and outbox persistence to `common.py`, Activity domain writes to `activity_repository.py` and worker/consumer files, and Geodata relational/import retention to `relational_state.py` and `import_retention.py`. Geodata `source_snapshot_*` is active import provenance; the generic `service_state` row is a legacy archive. Core normalized compatibility relations remain preserved until the full object-to-callsite audit is complete.
- **Source guide:** Activity PR #9 corrected the stale eight-step migration instructions and merged to `testing` at `1a4a8fcf0cf0bcf2a34ac27f36050f9ea08384c2`; Python quality run 38376070414 passed.
- **Issue status:** Deploy #15 has 5/7 acceptance checks marked, Activity #4 has 4/6, and Geodata #8 has 5/7. Paired rebuild issues #16/#5/#9 remain open with preliminary pre-quiescence archive evidence only. No live database was dropped or modified.
- **Live deployment:** Helm 282 is a rollback to 281. Fleet tracks Deploy `main` commit `9d80741`; its migration job ran the historical chain. The fixed runner remains on `testing` and is not in the live image.
- **Canonical record:** [Database baseline and rebuild inventory](../operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Database baseline owner inventory follow-up

- **Prompt used:** Continue the database migration baseline and data-preserving rebuild issues under the standing `AGENTS.md` workflow.
- **Scope:** Map the historical Core, Activity, and Geodata migration chains to the consolidated targets; document current table groups and exact source paths. This does not authorize a live rebuild.
- **Evidence:** Core's historical Operations migrations created the JetStream and SeaweedFS sampled-history tables; the forward retirement migrations removed them, and neither object is present in the current Core baseline. Activity and Geodata histories are individually listed in the canonical inventory. Generic `service_state` and Geodata `source_snapshot_*` object ownership remains explicitly unresolved pending call-site evidence.
- **Status:** The source/mirror baseline work and runner checks are verified on `testing`. Core, Activity, and Geodata baseline issues remain open for the exact retained-object owner audit and focused service checks. Paired rebuild issues remain gated on a deployed runner and final post-quiescence backup/restore rehearsals.
- **Record:** [Database baseline and rebuild inventory](../operations/databases/baseline-migration-inventory.md).

## 13 October 2026 — Database baseline runner verification and issue tracking

- **Prompt used:** “Work on the issues under ‘Database migration baselines and data-preserving rebuilds’ in the organization issue index to completion following the standing instructions in `./AGENTS.md`.”
- **Scope:** Continue the three schema baselines, synchronized Deploy/Platform mirrors, unified runner, and paired data-preserving rebuild issues. No live database reset was performed.
- **Source work:** Core, Activity, and Geodata baseline PRs are merged to `testing`; the Platform mirror is synchronized. Runner rehearsal found that PostgreSQL 16.4 rejected pg_dump's `\\restrict` / `\\unrestrict` psql metacommands in the Core and Activity baselines. Removed those directives from the owning and mirror files and merged the fix to `testing` in Deploy #19, Activity #8, and Platform #4.
- **Runner evidence:** In isolated `codex-db-rebuild` databases, the fixed runner applied each baseline once, repeated without duplicate ledger rows, and set readiness only after success. Hiding `public.account` caused the runner to fail and kept all three readiness rows false; restoring the sentinel/checksum let a subsequent run complete. Runner test databases/jobs/configmap were removed afterward.
- **Issue updates:** Baseline issues Deploy #15, Activity #4, and Geodata #8 now have verified acceptance checkmarks and updated evidence. Paired rebuild issues Deploy #16, Activity #5, and Geodata #9 now record the preliminary archive checksums and isolated restore results. They remain open because those archives predate quiescence and the live Helm migration job still uses the historical chain.
- **Live boundary:** Helm revision 282 is a rollback to revision 281. Fleet tracks Deploy `main` commit `9d80741`; `myota-migrations-281` completed the historical migration chain. All live databases and readiness markers remain intact. No cutover go/no-go was recorded.
- **Checks:** Activity quality run 38374446912, Deploy quality run 38374441554, and Platform CI run 38374443683 passed. The full runner fresh/repeat/failure/recovery rehearsal passed using the PostgreSQL 16/PostGIS clients in the local K3s test namespace.
- **Current evidence:** [Database baseline and rebuild inventory](../operations/databases/baseline-migration-inventory.md).

## 12 October 2026 — Structured logging correlation and acceptance recheck

- **Prompt used:** “Work on and close all issues under ‘Structured logging and
  request-to-worker correlation’ in the organization issue index as instructed
  in `./AGENTS.md`.”
- **Scope:** closed the Programme, Activity, and Geodata producer-owned
  correlation issues after live evidence; repaired and deployed the Activity
  NATS header-access crash; added production OTLP identity to shared workers;
  linked bounded alerts to the operator runbook. Deploy issue #6 remains open
  for authenticated Grafana navigation, full Compose runtime parity, and
  representative capacity/recovery evidence.
- **Evidence:** the sanitized Programme fact reached `MYOTA_EVENTS` with its
  request correlation and has no consumer group. Activity and Geodata jobs
  completed with matching correlation in Loki, durable ACK floors advanced,
  and probe database/object/queue rows were removed. A unique synthetic path
  marker was absent from the normalized Geodata error log.
- **Loki review:** the live indexed label set contains only
  `deployment_environment`, `service_name`, and `service_namespace`.
  Correlation/trace/request IDs remain structured metadata. The live config
  enables `2w` retention; compactor metrics showed 119 successful retention
  applications. This does not claim actual 14-day expiry or representative
  capacity.
- **Alert links:** Deploy
  [7cd6ac2](https://github.com/myota-platform/myota-deploy/commit/7cd6ac233715ceedae25dd049441ed0ce080bee5)
  and Platform
  [07eeb86](https://github.com/myota-platform/myota-platform/commit/07eeb863330223f426234b32d3078a044d8f65f3)
  add the shared runbook URL to 33 Prometheus rules and two Grafana API alerts.
  Fleet's bundle is Ready at 71/71 resources and the 23 Deployments / 27
  replicas are ready. Helm revision 278 remains `pending-upgrade`, so this
  snapshot does not claim Helm release reconciliation.
- **Remaining gates:** no authorized browser session was used to exercise
  Grafana navigation. Docker and Podman are absent on the host, so Compose
  runtime/restart parity was not executed. No representative load or actual
  14-day expiration test is claimed. See the
  [12 October evidence](../observability/evidence/logging-implementation-2026-10-12.md)
  and [Deploy #6](https://github.com/myota-platform/myota-deploy/issues/6).


### 12 October rollout correction

The runbook-link rollout exposed a malformed Prometheus annotation that passed
Helm rendering but failed live Prometheus rule loading. Deploy commit 4910201 and Platform commit 5555541 fix the Prometheus YAML;
Deploy commit b460a69 fixes Grafana alert YAML. Deploy Python quality runs
38217037536 and 38218406097 and Helm chart validation run 38217036883 passed.
PyYAML checks pass all six Prometheus/Grafana source/chart copies. Helm revision 279 rolled back to revision 277 source and restored
Prometheus. The corrected revision 280 ConfigMap is now loaded: Prometheus
validated 33 rules, Grafana mounted two alert runbook links and three Logs /
Failures dashboard links, and Grafana logged successful dashboard provisioning.
All 23 Deployments / 27 replicas are ready. Helm 280 remains `pending-upgrade`;
Fleet reports `WaitApplied` 0/1 and sync generation 92 versus desired 94. The correction and exact release state are in the
[12 October evidence](../observability/evidence/logging-implementation-2026-10-12.md).

## 12 October 2026 — Dashboard enhancement issue closeout recheck

- **Prompt used:** “Complete and close Deploy issues #11, #12, and #13 as
  instructed in `./AGENTS.md`.”
- **Scope:** verify the Deploy-owned K3s workload, application workflow, and
  PostgreSQL/PostGIS dashboards in the current Helm/Fleet deployment; confirm
  issue status; update detailed issue comments and Docs evidence. Deploy owns
  the source and Platform is its synchronized Helm mirror.
- **Finding:** all three issues were already closed as completed, with their
  implementation and original live Prometheus query evidence recorded in the
  issue bodies and
  [dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md).
  No dashboard source changes were needed.
- **Live check:** Helm chart `myota-0.2.20`, revision 275, is deployed. Fleet is
  Ready, Deployed, and Monitored at Deploy commit
  [`c7b6956`](https://github.com/myota-platform/myota-deploy/commit/c7b6956ebd23a220eb359a9814d749d2b2d94b58),
  with 71/71 resources ready. All 23 Deployments and 27/27 replicas are Ready.
  Grafana is 1/1 Ready and its dashboard ConfigMap contains all three dashboard
  assets. No test workload or data was created.
- **Documentation and issue records:** the evidence page now distinguishes
  its original live-query checks from this later deployment recheck. Detailed
  criteria, accepted metric gaps, and links to the evidence were added to
  [Deploy #11](https://github.com/myota-platform/myota-deploy/issues/11),
  [#12](https://github.com/myota-platform/myota-deploy/issues/12), and
  [#13](https://github.com/myota-platform/myota-deploy/issues/13). All three
  remain closed. The current Deploy main revision is c7b6956; dashboard sources
  were unchanged, so no new Helm upgrade was necessary.

## 12 October 2026 — Admin geodata details save follow-up

- **Prompt used:** “Revisit issue
  https://github.com/myota-platform/myota-admin-web/issues/2.”
- **Scope:** Admin Web management dialog and browser regression; Deploy-owned
  image pin and Platform mirror; Admin workflow guide; Docs guide, diagram,
  change log, and current-state record; local K3s rollout verification.
- **Finding and decision:** the earlier visibility fix put Website and
  Description near the dialog top, outside the Entity name and Entity
  categories fields. Users edited one logical details set but had to save its
  pieces separately. The fields now appear after Entity name and Entity
  categories in the Name & categories tab. One **Save details** action sends
  metadata through the existing versioned PATCH and categories through the
  existing versioned PUT, refreshing the entity version between requests.
  Partial failure is reported and a remaining category draft stays available
  for retry; no new API or database path was introduced.
- **Source and automated evidence:** Admin commits
  [d833f5d](https://github.com/myota-platform/myota-admin-web/commit/d833f5dc26ca351684222ec2971e3ea6e4470d4c)
  and [daecbdd](https://github.com/myota-platform/myota-admin-web/commit/daecbddb6537a71517bfc76049701fbd2b346d1e).
  Workflow [#38187013011](https://github.com/myota-platform/myota-admin-web/actions/runs/38187013011)
  passed 29 unit tests, type checking, production build, six browser tests,
  and image publication.
- **Deployment evidence:** immutable Admin image
  `sha256:23f4466f743d7f41d4a3549510926effe71e797aacc1a67c22aff85a6cafec54`
  is pinned in Deploy
  [3b6acef](https://github.com/myota-platform/myota-deploy/commit/3b6acef8e45e83e40f6283accfaf52c40902bdb8)
  and synchronized to Platform
  [c51a95e](https://github.com/myota-platform/myota-platform/commit/c51a95ed80a9dd7b702e0b6a3212572b5266bbef).
  The Admin Deployment observes generation 41 and is Ready 1/1; the served
  bundle contains the Save details control and field labels in the expected
  order. All 23 Deployments are ready.
- **Release status:** Helm revision 274 is deployed and Fleet reports Ready with 71/71 resources
  ready. All 23 Deployments and 27/27 replicas are ready. No test data was
  created.
- **Documentation and issue:** the Admin workflow guide, Docs editor guide and
  diagram were updated. The complete implementation steps and evidence are in
  [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2),
  which is closed after live verification.

## 11 October 2026 — Admin entity metadata visibility follow-up

- **Prompt used:** “Revisit issue
  https://github.com/myota-platform/myota-admin-web/issues/2.”
- **Scope:** Admin web management dialog, Admin browser regression, Deploy and
  Platform image pins, Docs workflow guide/diagram, local K3s read-only rollout
  verification, and issue closeout.
- **Finding and change:** Website and Description were present but nested under
  Name & categories. A dedicated metadata section now appears near the top of
  the management dialog. Browser checks assert visibility and the persisted
  values.
- **Verification:** Admin workflow
  [#38183593814](https://github.com/myota-platform/myota-admin-web/actions/runs/38183593814)
  passed production build, type checking, publishing, and all 24 browser tests.
  The live Admin Deployment runs image
  `sha256:ac82ae5420a5a449a2c5a812d972e62f3f8a291edbae3e7a3f45695388abdb24`.
  Helm 0.2.20 revision 273 is deployed; Fleet reports Ready and 23 Deployments
  with 27 replicas are Ready. A read-only live asset inspection found the new
  section and input labels. No test data was created.
- **Commits:** Admin UI
  `c4314ba2eef69877219861e24c1220f89801ae3b` (browser assertions) and
  `0c1d0cc9a8262745fadd35c7c6fb14ca8737b5fb` (workflow documentation);
  Deploy `0dd7de270a111adf2169601c1e8e796ef203fc57`; Platform mirror
  `9cabc46d657e387b5adb2152c294af206dd0342f`.
- **Result:** Issue #2 was closed after the live image and rollout checks.
  The [entity editor guide](../domain/administration/entity-catalogue-editor.md)
  and [diagram](../architecture/diagrams/entity-catalogue-editor.md) describe
  the prominent section.

## 11 October 2026 — Operations Service retirement and OpenTelemetry migration

- **Prompt used:** “Work on and complete all issues related to Operations
  Service retirement and OpenTelemetry migration in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- **Scope:** Operations, Deploy, Contracts, Admin UI, Identity, Platform mirror,
  Docs, and local K3s. Repository archive remains pending in issue #3.
- **Decisions:** direct least-privilege Collector receivers with query capture
  disabled; internal plaintext because DB images do not enable TLS; Grafana
  authorization remains in Gateway and Identity.
- **Result:** Operations is absent from runtime, API clients, deployment
  manifests, image publishing, and Operations-only schema.
- **Verification:** Helm 0.2.20 revision 272 is deployed and Fleet Ready with
  23 Deployments and 27 replicas. Collector metrics for all databases are
  non-empty; auth failures stopped after credential normalization and role
  synchronization. Gateway auth returns 401 and `Cache-Control: no-store`
  without credentials. See
  [retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md).

Warning: truncated output (original token count: 26439)
Total output lines: 1515

# MyOTA implementation timeline

This timeline reconstructs the major implementation milestones from the local
Git histories of the service, client, contract, deployment, and integration
repositories available through 11 October 2026, cross-checked against the current
architecture and operations documentation. It is intentionally a concise
history of meaningful system changes, not a complete commit-by-commit changelog.
Dates are repository commit dates. A commit link points to a representative
change; the linked repository history contains related follow-up fixes.

The architecture evolved quickly during this period. Earlier entries describe
the state at that point in time; later entries may replace an earlier design.
The [repository map](../architecture/repository-map.md) and [architecture](../architecture/overview.md)
describe current ownership and are authoritative for the present-day system.

## 11 October 2026 — Observability dashboard enhancements

- **Prompt used:** “Work on and close all issues related to Observability
  dashboard enhancements in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- **Scope:** completed Deploy issues #11–13 across the Deploy-owned Grafana
  and Helm configuration, synchronized Platform mirror, and Docs evidence.
  No application runtime code changed.
- **Result:** deployed K3s workload/capacity, application workflow, and
  PostgreSQL/PostGIS dashboards. The K3s panels use namespace-scoped
  kube-state-metrics and private TLS-verified Kubelet scraping. Unsupported
  per-durable JetStream delivery and PostgreSQL maintenance metrics are
  disclosed.
- **Verification:** Helm 0.2.20 revision 265 deployed; Fleet's MyOTA bundle
  was deployed and monitored; 24 Deployments and 28 replicas were ready.
  Prometheus returned live K3s, application workflow, database, and Surveyor
  series. Deploy/Platform dashboard blobs match. See the
  [dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md).

## 11 October 2026 — Geodata Phase 5 owner-directed closure

- **Prompt used:** “Assume the phase 5 gates passed and close it.”
- **Disposition:** closed Phase 5 by owner acceptance of the remaining
  qualification risk. No new test evidence was supplied; therefore the record
  does not present unrun tests as passed, and unverified checklist items remain
  unchecked under the evidence rule.
- **Accepted gaps:** broad-catalogue and sustained-load performance,
  independent-row writes, node/storage failure and multi-node scheduling, a
  new-image canary, rollback/import recovery/queue redrive, and API scaling
  beyond three replicas.
- **Constraint:** the current deployment remains at its tested two-to-three
  replica range. Reopen the relevant gates before expanding capacity or
  changing stateful topology. No runtime or deployment changes were made.
- See the [owner-acceptance closure record](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md)
  and [Geodata scaling roadmap](../geodata/horizontal-scaling-roadmap.md).

## 11 October 2026 — Geodata Phase 5 scale qualification follow-up

- **Prompt used:** “Finish the geodata scale qualifications.”
- **Scope:** ran a bounded read-only 2/10/50-VU check against the current
  ten-entity catalogue, added a bounded independent-row edit profile and its
  local harness tests, and reconciled the open-gate indexes. Runtime code,
  deployment settings, and database state were not changed.
- **Evidence:** all read profiles passed with 0% request failures and 100%
  checks; the 50-VU run reached 95.07 requests/s with 22.94 ms p95. The
  Geodata API remained 2/2 Ready, its worker 1/1 Ready, and HPA at 2 replicas
  after the run. No fixture or test data was created.
- **Qualification limits:** the catalogue contained only ten entities, so
  these read results do not establish broad capacity. Independent-row writes,
  large-catalogue performance, a new-image canary, rollback and node/storage
  failure remain open. The live write credentials were unavailable, and the
  single-node stateful failure tests remain unsafe on this host.
- **Harness verification:** 18/18 local Node tests passed for the new
  independent-edits setup, row assignment, bounds, and exact-run cleanup.
  Geodata Python quality (run 108) and image build/publish (run 126) passed.
  No runtime image was deployed because the change only touched load-test
  tooling; the live Deployment remains on its recorded image digest.
- See the [Phase 5 evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md)
  and [scaling roadmap](../geodata/horizontal-scaling-roadmap.md).


## 11 October 2026 — Cross-repository architecture review

- **Prompt used:** “Review the entire architecture. Check the individual
  repositories. Identify essential flaws, duplicate and redundant code and API
  endpoints, unnecessary complexity in all repos and services, and create a
  REVIEW.md document with inconsistencies and a phased mitigation plan, tasks,
  and exit criteria ready for Codex implementation. Identify unfinished phases
  in other work in progress and update the To do and other relevant Docs pages.”
- **Scope:** reviewed current main sources for Identity, Programme, Activity,
  Geodata, Operations, Contracts, Deploy, Platform, Docs, both web clients, and
  the shared .github workflow. Service repositories remain authoritative;
  Deploy and Platform runtime/contract copies are treated as mirrors.
- **Findings:** recorded a duplicate destructive Geodata cleanup path in the
  canonical OpenAPI document, Operations source/runtime mirror drift affecting
  the PostgreSQL sampler, retired NATS paths still advertised by OpenAPI,
  duplicated shared helpers, a large Geodata API module, and unfinished
  Geodata scaling, logging, API consolidation, Programme configuration, and
  metrics-coverage work. Runtime impact of the source/image discrepancy was
  not independently verified.
- **Plan and status at that review point:** added
  [REVIEW.md](../../REVIEW.md) with proposed phases, individual tasks, exit
  criteria, and a self-contained prompt for each phase. Mitigation had not yet
  started at the time of that read-only review. The original review changed no
  runtime code, API, database, or deployed state and ran no tests. The later
  completion of Phases 1–4 is recorded in the next entry.



## 11 October 2026 — Cross-repository review mitigation, Phases 1–4

- **Prompt used:** “Fix REVIEW.md phases 1–4 iteratively. Make sure to retire
  all documentation in this repo and elsewhere for removed API endpoints and
  functionality. Use own criteria and general best practices, don't ask me.
  Document all work done in docs repo and other repositories as well. Leave a
  clean and tidy documentation everywhere.”
- **Phase 1:** added the Contracts-owned source/mirror manifest and CI audit.
  It checks 44 source files, 92 file mirrors, and five exact tree mirrors.
- **Phase 2:** established Platform's canonical logging source and exact
  synchronization; retained service-specific HTTP, persistence, upload,
  authorization, and metrics adapters with their owners.
- **Phase 3:** extracted Geodata route registration into catalogue, import,
  and job groups while keeping state, handlers, transactions, and data
  ownership in Geodata.
- **Phase 4:** removed unused pre-live aliases and their runtime headers,
  registries, and counters. The current contract has 132 operations and 170
  route registrations with no missing or unregistered routes or duplicate
  operation IDs. Audited lifecycle commands remain.
- **Evidence:** source revisions and passing workflow runs, Fleet
  Ready=True at Deploy f7d2e65, Helm revision 246 deployed, and Deployment
  readiness are recorded in the
  [phase evidence report](../architecture/evidence/review-phases-1-4-2026-10-11.md).
  Phase 0 closure is recorded in the following timeline entry.

## 11 October 2026 — Cross-repository review Phase 0 contract safety

- **Prompt used:** “Decide how to resolve: Phase 0 remains open for
  duplicate-YAML-key rejection and the final handler/mirror check.”
- **Decision:** use a standard-library Psych AST validator in Contracts CI.
  Self-test that duplicate keys in one mapping fail and same-named keys in
  separate mappings pass. Merge the two cleanup path definitions into one
  operation that matches the Geodata handler, including the `lt-` identifier
  constraint, both supported global-admin roles, environment and production
  opt-ins, exact confirmation, active-work and linked-data guards, bucket
  restrictions, and all response fields.
- **Changes:** Contracts owns the canonical OpenAPI and validator; its root
  copy and the Platform bootstrap mirror are synchronized byte-for-byte. The
  regenerated route inventory records 131 operations and 170 service route
  registrations. No runtime handler or deployed behavior changed.
- **Evidence:** Contracts commits
  [1ed6efb](https://github.com/myota-platform/myota-contracts/commit/1ed6efb476879da3d10abb20325a104e5f504e24)
  and
  [36cf244](https://github.com/myota-platform/myota-contracts/commit/36cf244bedaf6071e50d7c5aef973ccd890128c4)
  pass [CI run 38157218999](https://github.com/myota-platform/myota-contracts/actions/runs/38157218999).
  Platform mirror commit
  [2426e4d](https://github.com/myota-platform/myota-platform/commit/2426e4da186aca49cfa00a58a83746f3573cdf66)
  passes [CI run 38157179490](https://github.com/myota-platform/myota-platform/actions/runs/38157179490).
  The [Phase 0 evidence record](../architecture/evidence/review-phase-0-2026-10-11.md)
  contains the exact changed paths and results.


## 11 October 2026 — Structured logging rollout, Phases 1–5

- **Prompt used:** “Implement observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- **Phase prompt basis:** the plan's [Phase 1](../observability/prompts/logging-phase-1-shared-foundation.md),
  [Phase 2](../observability/prompts/logging-phase-2-loki-platform.md),
  [Phase 3](../observability/prompts/logging-phase-3-http-tracing.md),
  [Phase 4](../observability/prompts/logging-phase-4-workers-nats.md),
  and [Phase 5](../observability/prompts/logging-phase-5-grafana-correlation.md)
  prompts; Phase 6 and 7 acceptance requirements were reviewed.
- **Follow-up prompts used:** “Continue with the implementation of phase 3,
  decide any blocker based on best criteria and established best practices,
  verify exit conditions.” The latest prompt for this work was: “Implement
  observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- **Implementation and decision:** fixed the four service runner entrypoints
  that bypassed shared logging initialization, published and deployed their
  successful CI image digests, and confirmed all application/worker replicas
  Ready. Geodata boundary, load-harness, and upload-recovery checks passed.
  Loki stores OTLP event strings as the log body and attributes as structured
  metadata; the Grafana JSON parser and JSON-regex trace link did not match this
  representation. Deploy and Platform now query metadata directly and use a
  label-type `trace_id` derived field.
- **Live evidence:** a non-mutating Identity `/healthz` request produced a
  matching Loki log and Tempo `GET /healthz` span; live LogQL dashboard query
  shapes succeeded. No business data was created. Helm revision 225 mounted the corrected datasource/dashboard, and Grafana
  logged successful provisioning. All seven mounted dashboard queries returned
  successfully against live Loki. Fleet later returned to Ready=True on Deploy commit `904bef2`; Helm revision
  227 is deployed after rollback to 226. Authenticated UI links in both
  directions have not yet been verified.
  Phase 6 retention/growth/capacity/privacy and Phase 7 PostgreSQL/PostGIS work
  remain incomplete. See [logging evidence](../observability/evidence/logging-implementation-2026-10-11.md),
  [phase prompts](../observability/prompts/README.md), and
  [implementation plan](../observability/logging-implementation.md).

## 11 October 2026 — NATS migration Phase 6 fact-stream cutover

- **Prompt used:** “Finish implementing phase 6 of
  https://github.com/myota-platform/myota-docs/blob/main/docs/operations/messaging/nats-event-migration-plan.md.
  No metrics, no shadowing, no canary. Usual decision-making and
  documentation requirements and consistency.”
  The copyable, self-contained execution prompt is recorded in the Phase 6
  section of the [migration plan](../operations/messaging/nats-event-migration-plan.md#phase-6).
- **Decision and implementation:** Used one direct gated cutover. The
  Deploy-owned Helm pre-upgrade migration changed the empty, legacy
  Interest-retained `MYOTA_EVENTS` to finite Limits retention with only
  `myota.events.>`. Its one-time migration setting is disabled after cutover;
  topology drift validation remains enabled. All three database-specific relay
  copies now enforce the fact-only stream, while Activity and Geodata work stay
  on separate WorkQueue streams. No runtime service business logic changed.
- **Evidence:** Preflight found zero stream messages/bytes, a drained Activity
  durable and owner outboxes, and preserved six unresolved Geodata DLQ rows.
  The live stream caps are Events 1 GiB/500,000/1 MiB, Activity work
  1 GiB/250,000/1 MiB, and Geodata work 3 GiB/500,000/1 MiB, each with 30-day
  max age, file storage, one replica, and DiscardNew. The disposable K3s test
  verified migration refusal/target validation and idempotent rerun without
  publishing a message; its namespace was deleted.
- **Rollout:** Deploy focused Python quality and Helm validation passed
  (runs 38099669717 and 38099669405). Helm revision 213 is deployed with the
  migration flag off. Fleet is Ready=True at Deploy main commit
  `dfc423d26bd79a82b1177b9e5d51e2d1196580c9`; every MyOTA Deployment has its
  desired replicas ready. No shadowing, canary, new metrics baseline, or
  production test data was used. No Phase 6 database object was obsolete.
- **Limits:** Previously Interest-expired facts remain unrecoverable from NATS.
  PostgreSQL remains authoritative and JetStream is only a bounded transport
  and replay window. Off-node broker recovery is still deferred. See the
  [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md),
  [ADR-0008](../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md),
  and [live topology diagram](../architecture/diagrams/nats-event-migration.md).

## 11 October 2026 — NATS Surveyor retirement implementation

- **Prompt used:** “check implementation of
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/nats-surveyor-migration.md,
  implement missing parts, update documentation, architecture and any other
  repos with dependencies.”
- **Service changes:** Operations removed JetStream inspection while retaining
  storage status and Grafana identity; Admin removed the old page, and Geodata
  removed duplicate broker polling. CI passed after fixing the Operations
  startup/formatting and removing the obsolete browser assertion.
- **Deployment decisions:** Surveyor 0.9.11; ten upstream dashboards pinned at
  `0539d22d7bc7a4574c7f376f1dde37f00bfe2ee9` plus adapted dashboard 16256;
  15-second scrape; existing resource limits; 15-day Prometheus retention;
  JetStream caps 7 GiB file / 1 GiB memory; warning/critical capacity alerts
  at 80/90% for ten minutes. Keep monitoring cluster-internal.
- **Recovery and live checks:** The initial Helm render emitted scientific-
  notation byte caps and NATS rejected the replacement while the old server
  remained healthy. Revision 204 restored the prior release; decimal-string
  limits and `Recreate` were then applied. Helm revision 208 is deployed with
  7 GiB/1 GiB limits and the replacement Operations, Admin, and Geodata images.
  Grafana lists eleven NATS dashboards; a synthetic in-memory stream was
  observed and deleted. Operations migration 003 / core migration 005 removed
  the snapshot table/index after the verified external backup gate; storage
  history remains. Fleet is Ready=True on Deploy main commit
  `d0d2f50846fe2c9e239d46a48809860f4b98d7f5`, with 64/64 resources ready and
  matching applied/desired IDs; Phase 2 is complete. See the
  [monitoring evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md)
  and [roadmap](../observability/nats-surveyor-migration.md).

## 11 October 2026 — Phase 5 early observation closure and retirement

- **Prompt used:** “Ok, complete the following phase 5 item now: Complete the
  24-hour rollback observation after immutable-image Helm revision 193, then
  recheck replacement/legacy filters, backlog counters, migration markers,
  owner-row recovery age and Fleet readiness. Retire only the four legacy
  Geodata durable definitions after this check. The earliest time is
  22:03:29 UTC on 11 October 2026. Ignore the earliest possible time, but
  conduct any additional tests needed to ensure this is safe, cleaning up
  test data afterwards.”
- **Timing boundary:** Fleet was Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a at 22:03:29 UTC on 10 October.
  At the user's explicit instruction, the 24-hour elapsed-time requirement was
  waived and checks closed at approximately 22:13:48 UTC on 10 October,
  about 10 minutes 19 seconds after the anchor. This is not a claim that a
  full 24-hour observation elapsed.
- **Final check and retirement:** Helm revision 193 remained deployed and
  Fleet and all MyOTA Deployments were Ready. Both streams were present and
  empty; the four old Geodata filters were exact with zero pending,
  ack-pending and redelivery; all four replacement durables retained exact
  filters, zero counters and one waiting pull. Activity notification
  remained. Migration 021 columns/indexes were present and the four
  age-bounded recovery queries had zero due rows. Deleted the four exact
  legacy Geodata consumers sequentially, verifying absence after each.
- **Post-check:** The shared event stream, Activity notification durable,
  Geodata work stream and all four target durables remained intact. A
  disposable in-memory broker test verified deletion semantics without
  publishing messages. Both Phase 5 disposable namespaces were deleted and
  verified absent; no production test data or broker messages were created.
- **Test limitation:** Two full delivery-suite attempts each exposed one
  timing-sensitive immediate ACK-counter assertion. Focused ACK verification
  passed and later private-stream counters settled at zero; the full suite is
  not claimed as a clean pass.
- **Status:** Phase 5 is complete within this evidence boundary and the
  explicit user waiver. Phase 6 fact-stream retention work is separate. See
  the [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md)
  and [migration plan](../operations/messaging/nats-event-migration-plan.md).


## 10 October 2026 — NATS migration Phase 5 Geodata cutover and recovery work

- **Prompt used for this phase:**

  ```text
  Continue work implementing phase 5 iteratively,, same criteria and instructions as last phase.
  ```

- Moved all four Geodata work consumers to the file-backed, bounded
  `MYOTA_GEODATA_WORK` WorkQueue and exact pull subjects/durables. Production
  workers use target subjects; the four legacy Geodata durables remain empty
  and inactive during the rollback observation. The Interest-retained
  `MYOTA_EVENTS` stream and Activity notification durable remain live.
- Applied Geodata migration 021 with owner-row dispatch timestamps and recovery
  indexes. Source rows and outbox remain authoritative. No Phase 5 database
  schema, source rows, history or dead letters were removed.
- The partial-deletion recovery fix keeps failed Geodata work retryable after an
  Activity cascade. The Activity handler/repository idempotency fix persists
  the request key and result transactionally. Five rounds of eight concurrent
  same-key requests each produced one cascade outbox fact.
- Isolated host-K3s checks passed: 154 Geodata tests (one optional skip), 40
  Activity tests (one optional broker skip), 25 relay/topology tests, actual
  database refusal/reconnect, same-node NATS PVC restart, two-database
  cascade-failure redelivery, cancellation racing acknowledged preprocessing,
  and the connected expiry → owner-row/outbox reconstruction → completion
  chain. The disposable namespace/PVC, test fixtures, API process and
  port-forwards were removed.
- A production inspection found that image digest settings changed only pod
  annotations while the container refs remained mutable `:latest` tags.
  Deploy now renders all configured first-party runtime, worker, provisioner
  and scheduled-job digests as immutable `repository@sha256:…` refs; Platform
  mirrors those templates. Helm revision 191 deployed at 21:28:41 UTC on
  10 October. Fleet reports Ready=True at Deploy commit `a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109` with
  60/60 resources. Activity, Geodata and shared-runtime pod refs and ImageIDs
  match the configured digests. Activity idempotency fix is live.
- Read-only broker verification found `MYOTA_GEODATA_WORK` file-backed,
  one-replica WorkQueue with finite limits and four exact pull durables; all
  target messages/counters are zero with active pull workers. `MYOTA_EVENTS`
  remains file-backed Interest retention with four inactive, empty legacy
  Geodata durables and its live Activity notification consumer.
- **Previous observation anchor (superseded):** Helm revision 192 completed at 21:39:22 UTC
  on 10 October after a chart release restarted the pods with unchanged image
  digests. Fleet is Ready=True at Deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a`, with 60/60
  resources. The 24-hour rollback observation now ends no earlier than
  21:39:22 UTC on 11 October. Recheck topology, recovery and Fleet, then retire
  only the four old Geodata durables. A read-only 21:52 UTC sample confirmed
  both streams healthy, zero work messages, zero legacy pending/ack-pending/
  redelivery, and waiting replacement workers. This early sample is not the
  24-hour gate; keep all four old durables. No database object is obsolete;
  migration 021 and authoritative records remain. Phase 6 remains separate.
- **Source commits:** Activity idempotency changes in
  `myota-activity-service` are handler `cdea2ba`, repository `d364932`,
  test `6448fc2`; mirrors are `6dbab48`/`9953b00` in Platform and
  `b76f825`/`259e2a7`/`84764e9` in Deploy. Image reference hardening:
  [Deploy `a68eedd`](https://github.com/myota-platform/myota-deploy/commit/a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109)
  and [Platform `a184baa`](https://github.com/myota-platform/myota-platform/commit/a184baac3f36e0272cbc79107f4b362139de7515).
  See [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md).

## 10 October 2026 — NATS migration Phase 3 domain-event consumers

- **Prompt used:** “Continue with the implementation of phase 3, decide any
  blocker based on your best criteria and established best practices, verify
  exit conditions.” This is the single self-contained Phase 3 prompt in the
  [migration plan](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-3).
- **Implementation:** Activity now consumes only the 21 registered Identity
  and Geodata fact subjects through stable durable `activity-notifications-v1`.
  The projection and deduplication/checkpoint commit atomically before ACK;
  diagnostic payloads are redacted; poison records have an audited redrive
  path; and outcome/DLQ metrics have bounded cardinality. Deployment hooks
  provisioned the exact successor before retiring the two broad Activity
  durables. The four Geodata work durables and mixed Interest-retained stream
  were unchanged.
- **Checks:** Activity CI and image publication passed. Deploy Python quality
  and Helm validation passed. Contracts CI initially failed only Ruff
  formatting on its new registry-filter assertion; the formatting fix passed
  the rerun. The 29-test Activity suite passed (one broker test skipped without
  `NATS_TEST_URL`); a separate disposable K3s/PostgreSQL/NATS drill verified
  duplicate/ACK-loss handling, poison redrive, metrics, and clean shutdown.
- **Live result and recovery:** On the local K3s cluster, the exact Activity
  durable is live with zero pending, ack-pending, or redelivered messages; one
  pull is waiting. Five stream durables remain: Activity plus the four
  unchanged Geodata consumers. The first migration hook used a stale pinned
  image digest; an additive idempotent Activity migration was applied by a
  short-lived repair Job, then reapplied successfully by normal migration Job
  `myota-migrations-169`. The repair Job was deleted. Helm revision 170 is
  deployed and Fleet reports 60/60 resources ready, but the Fleet bundle still
  reports `WaitApplied` after recovery. Phase 3's two documented exit criteria
  pass; resolve that Fleet status divergence before Phase 4 runtime changes.
- **Remaining scope:** The mixed `MYOTA_EVENTS` Interest stream, producer paths,
  Activity DB-polled work, and Geodata work paths remain as before. Payload
  privacy/schema enforcement, work-command migration, and final topology
  cutover remain open. See the [Phase 3 evidence](../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
  [consumer runbook](../operations/messaging/activity-notification-consumer.md),
  and [migration plan](../operations/messaging/nats-event-migration-plan.md).

## 10 October 2026 — NATS cluster-internal trust boundary decision

- **Prompt used:** “Remove the requirement for secure NATS / Jetstream because
  communication is cluster-internal. Then continue implementation of phase 1”.
  This is a Phase 1 decision follow-up; the plan retains one copyable Phase 1
  implementation prompt updated to match this decision.
- **Decision:** The workspace owner superseded the earlier proposal to require
  NKey credentials, NATS authentication, and TLS. Keep NATS reachable only via
  the Kubernetes ClusterIP service and trust all pods that can reach it. Do not
  expose port 4222 through an external service, NodePort, LoadBalancer, host
  port, or Ingress. Revisit this if the network boundary or workload trust
  model changes. Operations remains read-only by application behavior; the
  unauthenticated broker does not enforce that distinction.
- **Evidence:** Read-only K3s inspection showed `myota-nats` as ClusterIP
  `10.43.85.18:4222`, no NetworkPolicy in namespace `myota`, and Helm release
  `myota` at revision 157. No NATS deployment, stream, consumer, or application
  runtime was modified for this decision.
- **Phase 1 status:** Removed authentication/TLS and per-role NKey rollout from
  the open work and exit gates. Capacity sizing, payload projection and unknown
  route enforcement, chart readiness/relay compatibility, and off-node restore
  and database reconciliation remain open. See the
  [migration plan](../operations/messaging/nats-event-migration-plan.md),
  [joint review](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  and [topology diagram](../architecture/diagrams/nats-event-migration.md).

## 10 October 2026 — NATS Phase 1 Work checklist follow-up

- **Prompt used:** “OK, complete the section "work" in phase 1 now based on the
  decisions in https://github.com/myota-platform/myota-docs/blob/main/docs/operations/messaging/evidence/phase1-joint-review-2026-10-10.md”.
  This continues the same Phase 1 prompt in the
  [migration plan](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-1);
  no new phase prompt was introduced.
- **Contract checks:** Added fixtures requiring every event to declare a
  consumer disposition and every selected work command to match its registered
  subject, stream, schema, and deploy-owned durable filter. The immutable event
  and work envelopes also require UUID IDs and UTC timestamps ending in Z. The
  Contracts CI workflow passed on commit
  [d57f6b5](https://github.com/myota-platform/myota-contracts/commit/d57f6b5a8f8206639daac0426d08b81d10ba8b0f)
  ([run 38046041494](https://github.com/myota-platform/myota-contracts/actions/runs/38046041494)).
  Three focused contract tests passed, and the five-service source audit
  verified 68 fact types and six legacy work types with no undispositioned
  event-like literals. The UTC work-contract documentation was synchronized in
  [contracts commit 4f6d33d](https://github.com/myota-platform/myota-contracts/commit/4f6d33d65d850f6b238bbc77cabda9be96ed250f)
  and [platform mirror commit 19303f8](https://github.com/myota-platform/myota-platform/commit/19303f8b90e2b64495613837787a5d930a6ee886);
  [Contracts CI](https://github.com/myota-platform/myota-contracts/actions/runs/38046227981)
  and [platform CI](https://github.com/myota-platform/myota-platform/actions/runs/38046230452)
  passed.
- **Recovery documentation:** Added the [JetStream recovery and replay runbook](../operations/messaging/jetstream-recovery.md)
  and linked it from the messaging index, migration plan, and backlog. The
  runbook records PostgreSQL authority, isolated restore/replay, work redrive,
  legacy Interest-retention limits, and remaining qualification requirements.
- **Phase status:** Marked only verified contract, registry, topology-definition,
  provisioning-test, and runbook tasks complete. Authentication/TLS and role ACLs,
  final capacity values, deployment preflight, unknown-route enforcement,
  off-node/PVC-loss recovery, and relay cutover remain open. The review decisions
  are complete; their evidence gates are not.
- **Host verification:** Read-only K3s and Helm inspection found the MyOTA release
  at revision 157 and the node Ready. No production resource or producer/consumer
  path was changed.

## 10 October 2026 — NATS Phase 1 delegated joint review

- **Prompt used:** “OK, condict the Joint review of all payload schemas,
  credentials, production capacity limits, restore/replay qualification, and
  relay-side provisioning on your own bsed on best practices and your best
  criteria and continue phase 1 implementation.” This continues the existing
  [single Phase 1 prompt](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-1);
  no new phase prompt was introduced.
- **Schema decisions:** Reviewed all 68 source-derived fact schemas and their
  classifications/dispositions. Accepted them as inventory contracts, with
  producer enforcement gated on minimal projections, prohibited-field/size
  checks, compatibility fixtures, and Geodata preprocessing minimization. The
  ten selected work-command payloads were dispositioned as pending source,
  idempotency, recovery, and size evidence.
  The v1 Geodata preprocessed event still includes internal _records and
  _status; a compact versioned summary is required before enforcement.
- **Security and topology decisions:** Selected per-role NKey credentials from
  operator-managed Kubernetes Secrets, NATS TLS, no shared runtime identity, a
  read-only Operations role, and a separate create-only provisioner identity.
  Relay-side topology mutation will be removed only at a reviewed compatibility
  cutover.
- **Capacity and recovery:** Proposed 1 GiB for facts, 1 GiB for Activity work,
  and 3 GiB for Geodata work, reserving 3 GiB of the existing 8 GiB PVC.
  Evidence queried from deployed relay pods had no pending outbox rows, but
  covered only 2–9 October and included load-test data. Values are not final
  until representative 30-day sizing and pressure/recovery tests pass. Selected
  off-node stream snapshots, isolated restore/replay, declarative consumer
  recreation, and database-authoritative work redrive.
- **Checks and boundaries:** Read-only K3s query showed the node Ready, release
  myota at Helm revision 157, NATS single replica with an 8 GiB PVC, and all
  three outbox databases had no unpublished rows. No messages, streams, consumer
  state, or domain data changed. Credential rollout, off-node/PVC-loss restore, measured sizing, chart
  provisioner readiness, payload enforcement, and relay mutation removal remain
  open. A disposable two-broker NATS 2.10 drill restored three messages and the
  durable ACK position; a fresh durable replayed all three. Temporary namespace,
  CLI, and backup were removed. No production service was modified. Phase 1 is
  not complete. See the
  [joint review evidence](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  [migration plan](../operations/messaging/nats-event-migration-plan.md),
  and [topology diagram](../architecture/diagrams/nats-event-migration.md).

## 9 October 2026 — NATS migration Phase 1 contract and provisioner preparation

- **Contract baseline:** Added a contracts-owned registry for 68 domain facts,
  six proposed Activity work commands, and four proposed Geodata work commands.
  Added event/work envelope schemas and generated per-event outer-envelope
  schemas. Payload contracts remain pending owning-service review.
- **Provisioning:** Added a create-only, drift-checking JetStream provisioner
  that requires explicit finite capacity values and refuses to change existing
  streams or durables. Added a disabled-by-default Compose profile and a
  provisioning runbook. No producer, consumer, or live stream changed.
- **Observed evidence:** Read-only inspection found the deployed single
  `MYOTA_EVENTS` stream still uses Interest retention and has no finite
  byte/message caps; the NATS PVC requests 8 GiB and server args show no auth
  configuration. Three outbox databases had 19,103 rows in the measured 30-day
  query window, but the rows span only 2–9 October and include test load, so
  they do not establish 30-day capacity. The Phase 1 evidence report records
  exact per-database figures and limits.
- **Verification and disposition:** Two contract tests and three topology
  tests passed; Python compilation and diff checks passed. An isolated host-K3s
  JetStream broker created all target streams and work durables, accepted an
  identical second provision, rejected changed capacity configuration without
  mutation, and passed a subsequent matching recheck. The temporary namespace
  was deleted. Live provisioning, restore/replay qualification,
  least-privilege credentials, payload schemas, and relay-side provisioning
  removal remain open. Phase 1 is not complete.
  See the [evidence record](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md),
  [plan](../operations/messaging/nats-event-migration-plan.md), and
  [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
  The contracts #2, deploy #4, platform mirror #1, and organization profile #1
  PRs are merged. Merge records are linked from the evidence report. This closes
  review status only; Phase 1 exit gates remain open.
- **CI follow-up:** Corrected Ruff formatting in the contracts and deploy
  implementations and synchronized platform mirrors. Contract checks, deployment
  Ruff formatting/lint, and platform tests plus Ruff formatting/lint pass. The
  platform image-build job timed out requesting a Docker Hub token on two
  PR-CI attempts, leaving its image status unverified at that point. The Phase 1
  prompt above remains the single prompt for this phase; latest main-branch image
  build and publish results are recorded below. On 10 October, the deploy-owned
  provisioner was hardened to pin and verify consumer replay, pull-wait, delivery,
  storage, and payload settings; four focused topology/provisioner tests passed.
  The next contract step derives payload shapes for all 19 Identity facts from
  `myota-identity-service/identity.py`, keeps additive fields accepted, marks
  data classification, and adds contracts CI assertions. This is source-derived
  evidence pending Identity owner/privacy review; payloads for the four other
  producer services remain open. The existing Phase 1 prompt was used; no new
  phase prompt or runtime migration was introduced. Contract CI passed on
  commit `99547de`; the synchronized platform mirror passed unit, Ruff, and
  container checks on `529631c`.
  A subsequent source pass derived schemas for all 12 Programme facts, including
  content and policy review histories. Untyped producer-controlled configuration/content fields
  remain unconstrained; review metadata is classified. The same Phase 1 prompt
  continues. Identity and Programme joint review and three other service payload
  schemas remain open.
  The Programme contract commit `677f8ae` passed Contracts CI, including schema
  assertions and Ruff; platform mirror commit `b8c8507` passed unit, Ruff, and
  container checks.
  The next source pass derived additive schemas and sensitivity classifications
  for all 10 Activity facts from `myota-activity-service/activity.py`,
  `myota-activity-service/activity_repository.py`, and
  `myota-activity-service/awards.py`. Flexible rule/configuration and nested asset
  shapes remain open; joint owner/privacy review is still required. Contracts
  commit `2b8bddaf` and platform mirror commit `bf6f2727` are on `main`. Local
  contract checks pass; Activity Contracts CI passed in
  [run 38040217352](https://github.com/myota-platform/myota-contracts/actions/runs/38040217352).
  The existing Phase 1 prompt was reused. No producer/consumer runtime path changed.
  Direct-main commits `1076584` (deploy) and `b047a00` (platform) passed Ruff,
  deploy gateway/service image builds, platform tests, container build, and publish.
  See the [deploy checks](https://github.com/myota-platform/myota-deploy/commit/107658467eb708981322649e448832e272daccbb/checks)
  and [platform checks](https://github.com/myota-platform/myota-platform/commit/b047a00e15ecc619e3589fffee37a1aa779ff59a/checks). This follow-up did not change
  a live stream or producer/consumer runtime path. The same Phase 1 prompt
  remained in use for the source audit follow-up below. The contracts main-branch
  source audit passed in CI run 38033202206; the platform mirror tests, Ruff, and
  container check also passed on commit `038cd90`.

**Prompt used** (the single copyable Phase 1 prompt in the plan; retained for the
10 October follow-up audit):

```text
Implement the selected Phase 1 NATS contract and topology decisions recorded in
myota-docs/docs/operations/messaging/nats-event-migration-plan.md and ADR-0008. First read
the Phase 0 inventory and repository ownership map. Do not expand scope beyond the
approved topology.

Review the Phase 0 evidence ownership and gates table. Contract, schema, registry,
and isolated provisioning work may proceed, but do not apply live stream changes
or change producer/consumer behavior until the relevant gap is closed or the
specified owners record the required time-bounded risk acceptance and recovery
plan.

Update the authoritative myota-contracts event documentation/schema and any required
owning-service helpers. Define the versioned envelope, stable message ID, subject
naming/registry, event versus work-command classification, consumer-group naming,
unknown-version behavior, payload bounds, correlation/causation fields, and backward
compatibility rules. Update synchronized myota-platform/myota-deploy contract copies
only through their documented sync process.

Make JetStream stream and durable-consumer provisioning deterministic and
drift-checked. Register every producer subject, but provision durables only for
independent consumer groups with an intended business use; do not create broad
no-op consumers for Limits-retained facts. Provision required consumers before
relying on their processing. Implement the single controlled provisioner specified
in ADR-0008; relay replicas must not concurrently mutate stream configuration.
Validate all correctness-sensitive consumer settings.
Add least-privilege credentials and deployment configuration for each relay/worker
role. Add registry/contract checks that fail when a producer subject lacks schema,
owner, consumer disposition, provisioning and documentation.

Document retention limits, restore/replay and recovery procedures; preserve
PostgreSQL as system of record and Operations as read-only broker inspection. Update
myota-docs and operator docs. Add fo…6439 tokens truncated…e public gateway health check
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


## 10 October 2026 — Geodata payload contract source pass

- Added additive, source-derived schemas and classifications for all 27 Geodata
  facts from `myota-geodata-service/geodata.py`. Registry tests assert the
  source-derived evidence, classifications, and representative payload fields.
  Full entity and deletion-job payloads remain additive because their nested
  records are source-owned and extensible.
- The source review exposed that `geodata.import.preprocessed.v1` currently
  receives the complete preprocessing result, including internal `_records` and
  `_status`; `_records` can contain imported source features. This requires
  payload minimization and Geodata owner/privacy review before producer
  enforcement. No runtime event path changed.
- Local checks passed: contracts tests 2/2, all 68 schema generation, workspace
  source audit, Ruff lint/format, and byte-identical contracts/platform mirror
  comparison. Contracts CI passed for commit `e34de821` in
  [run 38042564323](https://github.com/myota-platform/myota-contracts/actions/runs/38042564323);
  platform mirror CI passed for `63cec55e` in
  [run 38042622987](https://github.com/myota-platform/myota-platform/actions/runs/38042622987).
  Joint owner/privacy review remains open. The Phase 0 audit found no Operations
  event-producing call sites, so Operations payload schemas are not applicable
  unless that service becomes a producer. The existing Phase 1 prompt was
  reused; no new phase prompt or live topology change was introduced. See the
  [evidence record](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [Phase 1 plan](../operations/messaging/nats-event-migration-plan.md).


## 10 October 2026 — Phase 1 contract/topology closeout

- Added bounded per-command payload schemas for all ten selected work types,
  with owning database row IDs and producer/consumer source references. The
  Contracts suite passes 4/4 tests; the workspace source audit found no
  undispositioned Python event-like literals.
- Added the optional fail-closed Helm pre-upgrade provisioner hook and chart
  render checks. It remains disabled until the Phase 2 compatibility gate is
  confirmed. No live relay, producer, consumer, or `myota` namespace resource
  changed.
- Local K3s qualification verified provisioner create/idempotency/drift,
  replacement local PVC restore, bounded replay, and `DiscardNew` capacity
  rejection in `nats-phase1-verify-20261010`; the namespace and its resources
  were deleted. Off-node recovery is deferred at the project owner's direction.
  The short capacity sample and accepted 1/1/3 GiB starting caps are documented
  as bounded single-node decisions, not 30-day measured capacity.
- Deploy tests passed 36 tests with one Pillow-dependent skip. Helm lint passed
  for default and Spainip values; the optional pre-upgrade hook rendered only
  when explicitly enabled and confirmed. Docs, org profile links, and platform
  mirrors were updated.
- Post-push CI passed: [Contracts run 38051378258](https://github.com/myota-platform/myota-contracts/actions/runs/38051378258),
  deploy Python and Helm validation runs 38051333534/38051332975, and
  [platform mirror run 38051342419](https://github.com/myota-platform/myota-platform/actions/runs/38051342419).

**Prompt used for this Phase 1 continuation:**

```text
Address and work on the remaining Phase 1 tasks until completion, iteratively
if needed. If a decision is required, make it yourself based on your best
criteria and best practices.

Don't use the cloud-environment skill. Conduct work on the local K3s server in
a separate namespace and clean it up when done. Off-node recovery is not
required for the current scope.
```

The Phase 1 prompt in the [migration plan](../operations/messaging/nats-event-migration-plan.md)
was used as the implementation basis and updated to reflect these accepted
capacity, trust-boundary, recovery, and production-cutover decisions.

## 10 October 2026 — NATS Phase 2 relay hardening

- **Prompt used:**

  ```text
  OK. Go ahead and implement phase 2 iteratively and until exit criteria are complete.
  Take sane and safe decisions based on your best criteria and best practice without external review.
  ```

- **Relay and contract coverage:** Added generated runtime routing data from the
  canonical Contracts registry. The relay validates registered fact types,
  producer ownership, UUID identity, UTC timestamps, exact subjects, and the
  configured message-size cap. Facts use dotted registered subjects and stable
  `Nats-Msg-Id`; six current Geodata work source event types remain on their
  legacy subjects pending the controlled work-stream phase.
- **Failure and topology behavior:** Publish acknowledgement precedes the
  database mark. A failed mark leaves the row retryable with the same message
  ID; retry delay is capped, and terminal failures can be inspected and
  redriven from their source database with an audit record. Relay startup now
  validates legacy stream/durable settings read-only. The Geodata worker no
  longer removes a retired push durable at startup.
- **Retention and operations:** Added resolution/audit migrations for core,
  Activity, and Geodata dead letters. Geodata import cleanup preserves
  unresolved dead letters. Added per-relay backlog/retry/DLQ metrics, three
  scrape targets, alerts, and a dashboard; Operations remains metadata-only.
- **Verification:** The workspace audit verified 68 registered facts and six
  mapped legacy work types with no undispositioned event-like source literals.
  Contracts tests passed 5/5, deploy relay tests 17/17, and Geodata retention
  tests 6/6. The Activity duplicate-delivery regression passed 2 tests; its
  broker-backed overlap/restart test was skipped because it was not configured
  with an isolated broker. Ruff, Helm lint/render, Compose/Collector/alert/dashboard parsing,
  migration idempotency, live legacy metadata inspection, actual same-ID
  JetStream deduplication, publish/mark crash recovery, metrics, and audited
  redrive passed. The broker/database drill ran in a temporary K3s namespace
  that was removed after testing; no production broker or database object was
  changed during verification.
- **Boundaries:** The current production stream remains Interest-retained and
  mixed. Activity's broad notification durable and four Geodata work durables
  remain in place. Payload schema/prohibited-field enforcement, Geodata
  preprocessing minimization, consumer/work migration, production watermark
  comparison, and controlled cutover remain open. See the
  [Phase 2 evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
  [migration plan](../operations/messaging/nats-event-migration-plan.md), and
  [topology diagrams](../architecture/diagrams/nats-event-migration.md).
- **Rollout correction:**
  - Helm initially rendered the configured 1 MiB limit as `1.048576e+06`; the
    relay expects an integer string. Quoted the deploy-owned value and synced
    the platform mirror. Helm render and CI passed. A temporary live env update
    restored readiness until Fleet applied the quoted chart value.
  - The migrations Job next ran a cached image missing
    `dead_letter_event.resolved_at`. Updated its pull policy, versioned the
    chart as 0.2.14, and synchronized the current platform digest. Migration
    Job 166 then applied `resolved_at` and its unresolved index in all three
    databases.
  - All three relays are 1/1 Ready with healthy database/NATS gauges and zero
    pending rows. Geodata reports six unresolved DLQs, left untouched. Fleet
    is Ready with 59/59 resources, and Helm revision 167 is deployed. See the
    [Phase 2 evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md).

## 11 October 2026 — NATS Surveyor dashboard query verification

- **Prompt used:**

  ```text
  Check the implementation of
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/nats-surveyor-migration.md,
  implement missing parts, and update documentation, architecture, and any
  dependent repositories.
  ```

- Rechecked the live deployment on the host K3s cluster without writing test
  data. Grafana search returned eleven dashboards in folder `NATS`; all
  thirteen adapted dashboard 16256 expressions returned HTTP 200 and one data
  frame through Grafana's Prometheus datasource API, with no query error.
  Surveyor and NATS health were both 1, and no NATS alert was firing.
- Confirmed Deploy and Platform carry the same Surveyor alert rules (Git blob
  `f20ceece6fbf71e517a28008382f02d157c4560d`). No runtime source changes were
  required. The wider Prometheus snapshot had five separate Identity,
  Programme, and Geodata outbox alerts, recorded as follow-up outside this
  monitoring task.
- Recorded the sample values and alert boundary in the
  [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md)
  and updated the [monitoring roadmap](../observability/nats-surveyor-migration.md)
  and [observability architecture](../architecture/diagrams/observability-architecture.md).

## 11 October 2026 — Phase 7 database observability increment

- **Prompt used:** “Implement observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- Added the external-secret-backed monitor-role path, bounded PostgreSQL
  receiver configuration, and six-panel database dashboard in Deploy; Platform
  carries synchronized mirrors. Checked the monitor credentials through
  read-only queries to `pg_stat_database` across the three databases.
- The Collector receiver failed authentication with those same Secrets in both
  the live and disposable diagnostic Collector, while `psql` succeeded.
  Disabled the receiver pipeline to stop repeated failures. The deployed
  dashboard file parses, but its database data panels are not accepted.
- All Deployments report Ready. Helm revision 234 is deployed; Helm history labels it a rollback to revision
233. The Fleet BundleDeployment is Ready=True at sync generation 93. All 67
resources are ready, and all Deployments meet their requested replica counts. Phase 7 remains open; see the
  [evidence](../observability/evidence/logging-implementation-2026-10-11.md)
  and [plan](../observability/logging-implementation.md).

## 11 October 2026 — Structured logging, database signals, and deployed acceptance

- **Prompt used:** “https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md isn't complete. Decide on any gaps based on what you know and best practices. A transition or verification period isn't needed, as this is not yet a live system. Continue to implement the missing bits. No individual feile commits, one commit per task or item completed.”
- **Decision:** retain bounded PostgreSQL metrics through the Operations read-only adapter after the direct Collector receiver failed authentication. Keep PostgreSQL statement logs out of the telemetry pipeline because they may contain SQL literals and user data. Use Psycopg instrumentation with parameter capture disabled and Collector removal of SQL attributes.
- Operations database metrics, Psycopg spans, Loki capacity scrape/panels, and synced Compose/Helm source are deployed. Helm revision 241 is active with 67/67 Fleet resources ready. Live database metrics and SQL-free Tempo span attributes are documented in the [logging evidence](../observability/evidence/logging-implementation-2026-10-11.md).
- The roadmap remains open for authenticated Grafana navigation, request-to-worker correlation, Loki prohibited-field audit, and dynamic Compose runtime parity. No transition period was added; retention configuration was accepted by config and runtime checks without waiting 14 days.
## 11 October 2026 — Stable NATS logging correlation

- **Prompt used:** Continue the implementation of structured logging, decide on remaining gaps using best practices, and avoid transition or verification periods because the system is pre-live.
- Added a stable relay correlation fallback (`correlationId` → `workId` → `eventId`) to the NATS header and bound that value into consumer log context across the five owning services. Payload schemas remain unchanged.
- Service source CI, worker tests, Deploy validation, and Platform tests passed. Updated images are running and all Deployments are ready.
- The Helm release still reports revision 242 as `pending-upgrade` while Fleet reports revision 241 Ready. The originating HTTP correlation remains unavailable for outbox producers that did not persist it; live request-to-worker correlation is not claimed.

## 11 October 2026 — Current system and documentation reconciliation

- **Prompt used:** “Update the myota-docs repository with the current state of
  the system, including updated diagrams. Summarize evidence and task documents,
  and improve and simplify the overall documentation structure.”
- **Scope:** reconciled Docs against GitHub `main` heads for service, contract,
  deployment, platform, client, and organization repositories; inspected current
  MyOTA K3s Helm/Fleet/Deployment readiness read-only; corrected current-state
  status and architecture diagrams; consolidated root and subject navigation.
- **Evidence:** Helm `myota` revision 246 is deployed. Fleet reports Ready,
  Deployed, and Monitored with 67/67 resources at Deploy commit
  `f7d2e65edd5248d70376cf99ccc98c836b7f95e3`; 23/23 MyOTA Deployments were
  Ready. The single-node limitation and outstanding acceptance gates are
  explicit in [Current system state](../architecture/current-state.md).
- **Documentation decisions:** the new current-state page is the single dated
  snapshot; plans own checklists and dated evidence owns test results. Historical
  records remain snapshots. Canonical diagrams and subject indexes point to the
  snapshot. Compatibility pages remain for published links.
- **Runtime impact:** none. Checks were read-only; no tests, test data, or
  cluster changes were made.


## 11 October 2026 — SeaweedFS monitoring retirement

- **Prompt used:** “Re-read `AGENTS.md`, then work on, implement and close all
  issues related to SeaweedFS monitoring retirement.”
- Removed the Operations SeaweedFS sampler/API and Admin monitoring page, retired
  its sampled-history table with a forward-only migration, and retained only the
  application object-storage and readiness paths. Replaced the Grafana dashboard
  with a validated adaptation of dashboard 19173 and synchronized Platform copies.
- Updated the OpenAPI contract and generated clients, Admin legacy bookmark
  redirect, runbooks, repository ownership map, current-state summary, diagrams,
  README indexes and issue cross-links. Deploy remains authoritative for the
  Collector, Grafana provisioning, dashboard, alerts and Helm configuration.
- Evidence, release identifiers, live schema and endpoint checks, and any
  remaining environment limits are recorded in the linked Deploy, Operations,
  Contracts, Admin and Docs issues. Live workload, endpoint, database, and metric checks confirm the retirement paths.
  The Fleet/Helm revision status remains explicitly reported as pending while its
  applied-state metadata catches up; issue notes preserve that discrepancy.

## 11 October 2026 — Geodata TracerProvider warning repair

- **Prompt used:** “Work on and close the issues related to Geodata
  OpenTelemetry TracerProvider warning in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- Confirmed the provider was installed during startup by
  `myota_logging.configure_tracing()` and then installed a second time from
  `otel.telemetry_for()` on HTTP requests. Geodata now reuses one process-wide
  telemetry instance and its startup tracing provider. Setup failures remain
  visible without exposing endpoint values or interrupting request handling.
- Synchronized the Geodata runtime mirror to Deploy/Platform and the
  Platform-owned shared logger to its seven service/deployment mirrors. Focused
  tests, source CI, image builds, and live health/Tempo checks passed. The
  rollout reached Helm revision 257 with Fleet Ready 1/1 and all 23 Deployments
  ready (27/27 replicas). Geodata #7 and Deploy #14 were closed with verified
  acceptance criteria. The complete evidence is in the
  [TracerProvider repair record](../observability/evidence/geodata-tracerprovider-2026-10-11.md).


## 11 October 2026 — NATS migration evidence/readiness issue closeout

- **Prompt used:** “Work on and close all issues related to NATS migration
  evidence and readiness in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- Reviewed the Phase 0–6 evidence, authoritative Contracts registry, service
  dispositions, the three database relay paths, selected Activity/Geodata work
  consumers, and the deployment-owned provisioning and recovery boundary.
- Updated the Contracts registry status and tests for 68 facts; synchronized
  the registry mirror in Platform. The clean-clone registry suite passed 7/7,
  and the workspace event-source audit verified 68 facts and found no
  unclassified event-like literals. The Platform registry test job passed.
- Read-only K3s/NATS inspection confirmed all three disjoint streams and eleven
  expected consumers. Streams were empty, consumers had no pending work, three
  database relays were ready, Helm revision 257 was deployed, Fleet was Ready
  1/1, and all 23 Deployments (27/27 replicas) were ready. The temporary
  read-only NATS CLI pod was removed without publishing messages.
- Closed six source/contract/service/deployment issues with completion
  evidence. Closed Operations #1 as superseded by the current Surveyor and
  Operations retirement design. Refreshed the organization issue index.
- **Accepted limits:** the owner has deferred off-node recovery. Initial finite
  capacity ceilings are accepted for this single-node scope but are not a
  measured 30-day production qualification. No production event traffic is
  inferred from the zero current stream backlog. Six unresolved Geo DLQ
  records remain untouched.
- **CI caveat:** Contract CI run 38171573878 stopped at the mirror audit and
  reported Identity/Programme handler drift plus the Platform registry copy.
  The registry copy was synchronized afterward in Platform commit deb0d0e;
  a fresh direct audit now reports only the Identity/Programme handler drift,
  but the Contracts workflow has not been rerun after that sync. Platform run
  38171688089 passed its test job but failed Ruff formatting for
  `services/identity.py`. These results are recorded without claiming full
  repository CI success.
- Detailed issue mapping and runtime values are in the
  [NATS readiness closeout](../operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md);
  the [current system snapshot](../architecture/current-state.md) carries the
  reconciled deployment state.
## 12 October 2026 — Structured logging correlation and acceptance recheck

- **Prompt used:** “Work on and close all issues under ‘Structured logging and
  request-to-worker correlation’ in the organization issue index as instructed
  in `./AGENTS.md`.”
- **Scope:** closed the Programme, Activity, and Geodata producer-owned
  correlation issues after live evidence; repaired and deployed the Activity
  NATS header-access crash; added production OTLP identity to shared workers;
  linked bounded alerts to the operator runbook. Deploy issue #6 remains open
  for authenticated Grafana navigation, full Compose runtime parity, and
  representative capacity/recovery evidence.
- **Evidence:** the sanitized Programme fact reached `MYOTA_EVENTS` with its
  request correlation and has no consumer group. Activity and Geodata jobs
  completed with matching correlation in Loki, durable ACK floors advanced,
  and probe database/object/queue rows were removed. A unique synthetic path
  marker was absent from the normalized Geodata error log.
- **Loki review:** the live indexed label set contains only
  `deployment_environment`, `service_name`, and `service_namespace`.
  Correlation/trace/request IDs remain structured metadata. The live config
  enables `2w` retention; compactor metrics showed 119 successful retention
  applications. This does not claim actual 14-day expiry or representative
  capacity.
- **Alert links:** Deploy
  [7cd6ac2](https://github.com/myota-platform/myota-deploy/commit/7cd6ac233715ceedae25dd049441ed0ce080bee5)
  and Platform
  [07eeb86](https://github.com/myota-platform/myota-platform/commit/07eeb863330223f426234b32d3078a044d8f65f3)
  add the shared runbook URL to 33 Prometheus rules and two Grafana API alerts.
  Fleet's bundle is Ready at 71/71 resources and the 23 Deployments / 27
  replicas are ready. Helm revision 278 remains `pending-upgrade`, so this
  snapshot does not claim Helm release reconciliation.
- **Remaining gates:** no authorized browser session was used to exercise
  Grafana navigation. Docker and Podman are absent on the host, so Compose
  runtime/restart parity was not executed. No representative load or actual
  14-day expiration test is claimed. See the
  [12 October evidence](../observability/evidence/logging-implementation-2026-10-12.md)
  and [Deploy #6](https://github.com/myota-platform/myota-deploy/issues/6).


### 12 October rollout correction

The runbook-link rollout exposed a malformed Prometheus annotation that passed
Helm rendering but failed live Prometheus rule loading. Deploy commit 4910201 and Platform commit 5555541 fix the Prometheus YAML;
Deploy commit b460a69 fixes Grafana alert YAML. Deploy Python quality runs
38217037536 and 38218406097 and Helm chart validation run 38217036883 passed.
PyYAML checks pass all six Prometheus/Grafana source/chart copies. Helm revision 279 rolled back to revision 277 source and restored
Prometheus. The corrected revision 280 ConfigMap is now loaded: Prometheus
validated 33 rules, Grafana mounted two alert runbook links and three Logs /
Failures dashboard links, and Grafana logged successful dashboard provisioning.
All 23 Deployments / 27 replicas are ready. Helm 280 remains `pending-upgrade`;
Fleet reports `WaitApplied` 0/1 and sync generation 92 versus desired 94. The correction and exact release state are in the
[12 October evidence](../observability/evidence/logging-implementation-2026-10-12.md).

## 12 October 2026 — Dashboard enhancement issue closeout recheck

- **Prompt used:** “Complete and close Deploy issues #11, #12, and #13 as
  instructed in `./AGENTS.md`.”
- **Scope:** verify the Deploy-owned K3s workload, application workflow, and
  PostgreSQL/PostGIS dashboards in the current Helm/Fleet deployment; confirm
  issue status; update detailed issue comments and Docs evidence. Deploy owns
  the source and Platform is its synchronized Helm mirror.
- **Finding:** all three issues were already closed as completed, with their
  implementation and original live Prometheus query evidence recorded in the
  issue bodies and
  [dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md).
  No dashboard source changes were needed.
- **Live check:** Helm chart `myota-0.2.20`, revision 275, is deployed. Fleet is
  Ready, Deployed, and Monitored at Deploy commit
  [`c7b6956`](https://github.com/myota-platform/myota-deploy/commit/c7b6956ebd23a220eb359a9814d749d2b2d94b58),
  with 71/71 resources ready. All 23 Deployments and 27/27 replicas are Ready.
  Grafana is 1/1 Ready and its dashboard ConfigMap contains all three dashboard
  assets. No test workload or data was created.
- **Documentation and issue records:** the evidence page now distinguishes
  its original live-query checks from this later deployment recheck. Detailed
  criteria, accepted metric gaps, and links to the evidence were added to
  [Deploy #11](https://github.com/myota-platform/myota-deploy/issues/11),
  [#12](https://github.com/myota-platform/myota-deploy/issues/12), and
  [#13](https://github.com/myota-platform/myota-deploy/issues/13). All three
  remain closed. The current Deploy main revision is c7b6956; dashboard sources
  were unchanged, so no new Helm upgrade was necessary.

## 12 October 2026 — Admin geodata details save follow-up

- **Prompt used:** “Revisit issue
  https://github.com/myota-platform/myota-admin-web/issues/2.”
- **Scope:** Admin Web management dialog and browser regression; Deploy-owned
  image pin and Platform mirror; Admin workflow guide; Docs guide, diagram,
  change log, and current-state record; local K3s rollout verification.
- **Finding and decision:** the earlier visibility fix put Website and
  Description near the dialog top, outside the Entity name and Entity
  categories fields. Users edited one logical details set but had to save its
  pieces separately. The fields now appear after Entity name and Entity
  categories in the Name & categories tab. One **Save details** action sends
  metadata through the existing versioned PATCH and categories through the
  existing versioned PUT, refreshing the entity version between requests.
  Partial failure is reported and a remaining category draft stays available
  for retry; no new API or database path was introduced.
- **Source and automated evidence:** Admin commits
  [d833f5d](https://github.com/myota-platform/myota-admin-web/commit/d833f5dc26ca351684222ec2971e3ea6e4470d4c)
  and [daecbdd](https://github.com/myota-platform/myota-admin-web/commit/daecbddb6537a71517bfc76049701fbd2b346d1e).
  Workflow [#38187013011](https://github.com/myota-platform/myota-admin-web/actions/runs/38187013011)
  passed 29 unit tests, type checking, production build, six browser tests,
  and image publication.
- **Deployment evidence:** immutable Admin image
  `sha256:23f4466f743d7f41d4a3549510926effe71e797aacc1a67c22aff85a6cafec54`
  is pinned in Deploy
  [3b6acef](https://github.com/myota-platform/myota-deploy/commit/3b6acef8e45e83e40f6283accfaf52c40902bdb8)
  and synchronized to Platform
  [c51a95e](https://github.com/myota-platform/myota-platform/commit/c51a95ed80a9dd7b702e0b6a3212572b5266bbef).
  The Admin Deployment observes generation 41 and is Ready 1/1; the served
  bundle contains the Save details control and field labels in the expected
  order. All 23 Deployments are ready.
- **Release status:** Helm revision 274 is deployed and Fleet reports Ready with 71/71 resources
  ready. All 23 Deployments and 27/27 replicas are ready. No test data was
  created.
- **Documentation and issue:** the Admin workflow guide, Docs editor guide and
  diagram were updated. The complete implementation steps and evidence are in
  [Admin issue #2](https://github.com/myota-platform/myota-admin-web/issues/2),
  which is closed after live verification.

## 11 October 2026 — Admin entity metadata visibility follow-up

- **Prompt used:** “Revisit issue
  https://github.com/myota-platform/myota-admin-web/issues/2.”
- **Scope:** Admin web management dialog, Admin browser regression, Deploy and
  Platform image pins, Docs workflow guide/diagram, local K3s read-only rollout
  verification, and issue closeout.
- **Finding and change:** Website and Description were present but nested under
  Name & categories. A dedicated metadata section now appears near the top of
  the management dialog. Browser checks assert visibility and the persisted
  values.
- **Verification:** Admin workflow
  [#38183593814](https://github.com/myota-platform/myota-admin-web/actions/runs/38183593814)
  passed production build, type checking, publishing, and all 24 browser tests.
  The live Admin Deployment runs image
  `sha256:ac82ae5420a5a449a2c5a812d972e62f3f8a291edbae3e7a3f45695388abdb24`.
  Helm 0.2.20 revision 273 is deployed; Fleet reports Ready and 23 Deployments
  with 27 replicas are Ready. A read-only live asset inspection found the new
  section and input labels. No test data was created.
- **Commits:** Admin UI
  `c4314ba2eef69877219861e24c1220f89801ae3b` (browser assertions) and
  `0c1d0cc9a8262745fadd35c7c6fb14ca8737b5fb` (workflow documentation);
  Deploy `0dd7de270a111adf2169601c1e8e796ef203fc57`; Platform mirror
  `9cabc46d657e387b5adb2152c294af206dd0342f`.
- **Result:** Issue #2 was closed after the live image and rollout checks.
  The [entity editor guide](../domain/administration/entity-catalogue-editor.md)
  and [diagram](../architecture/diagrams/entity-catalogue-editor.md) describe
  the prominent section.

## 11 October 2026 — Operations Service retirement and OpenTelemetry migration

- **Prompt used:** “Work on and complete all issues related to Operations
  Service retirement and OpenTelemetry migration in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- **Scope:** Operations, Deploy, Contracts, Admin UI, Identity, Platform mirror,
  Docs, and local K3s. Repository archive remains pending in issue #3.
- **Decisions:** direct least-privilege Collector receivers with query capture
  disabled; internal plaintext because DB images do not enable TLS; Grafana
  authorization remains in Gateway and Identity.
- **Result:** Operations is absent from runtime, API clients, deployment
  manifests, image publishing, and Operations-only schema.
- **Verification:** Helm 0.2.20 revision 272 is deployed and Fleet Ready with
  23 Deployments and 27 replicas. Collector metrics for all databases are
  non-empty; auth failures stopped after credential normalization and role
  synchronization. Gateway auth returns 401 and `Cache-Control: no-store`
  without credentials. See
  [retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md).

Warning: truncated output (original token count: 26439)
Total output lines: 1515

# MyOTA implementation timeline

This timeline reconstructs the major implementation milestones from the local
Git histories of the service, client, contract, deployment, and integration
repositories available through 11 October 2026, cross-checked against the current
architecture and operations documentation. It is intentionally a concise
history of meaningful system changes, not a complete commit-by-commit changelog.
Dates are repository commit dates. A commit link points to a representative
change; the linked repository history contains related follow-up fixes.

The architecture evolved quickly during this period. Earlier entries describe
the state at that point in time; later entries may replace an earlier design.
The [repository map](../architecture/repository-map.md) and [architecture](../architecture/overview.md)
describe current ownership and are authoritative for the present-day system.

## 11 October 2026 — Observability dashboard enhancements

- **Prompt used:** “Work on and close all issues related to Observability
  dashboard enhancements in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- **Scope:** completed Deploy issues #11–13 across the Deploy-owned Grafana
  and Helm configuration, synchronized Platform mirror, and Docs evidence.
  No application runtime code changed.
- **Result:** deployed K3s workload/capacity, application workflow, and
  PostgreSQL/PostGIS dashboards. The K3s panels use namespace-scoped
  kube-state-metrics and private TLS-verified Kubelet scraping. Unsupported
  per-durable JetStream delivery and PostgreSQL maintenance metrics are
  disclosed.
- **Verification:** Helm 0.2.20 revision 265 deployed; Fleet's MyOTA bundle
  was deployed and monitored; 24 Deployments and 28 replicas were ready.
  Prometheus returned live K3s, application workflow, database, and Surveyor
  series. Deploy/Platform dashboard blobs match. See the
  [dashboard evidence](../observability/evidence/dashboard-enhancements-2026-10-12.md).

## 11 October 2026 — Geodata Phase 5 owner-directed closure

- **Prompt used:** “Assume the phase 5 gates passed and close it.”
- **Disposition:** closed Phase 5 by owner acceptance of the remaining
  qualification risk. No new test evidence was supplied; therefore the record
  does not present unrun tests as passed, and unverified checklist items remain
  unchecked under the evidence rule.
- **Accepted gaps:** broad-catalogue and sustained-load performance,
  independent-row writes, node/storage failure and multi-node scheduling, a
  new-image canary, rollback/import recovery/queue redrive, and API scaling
  beyond three replicas.
- **Constraint:** the current deployment remains at its tested two-to-three
  replica range. Reopen the relevant gates before expanding capacity or
  changing stateful topology. No runtime or deployment changes were made.
- See the [owner-acceptance closure record](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md)
  and [Geodata scaling roadmap](../geodata/horizontal-scaling-roadmap.md).

## 11 October 2026 — Geodata Phase 5 scale qualification follow-up

- **Prompt used:** “Finish the geodata scale qualifications.”
- **Scope:** ran a bounded read-only 2/10/50-VU check against the current
  ten-entity catalogue, added a bounded independent-row edit profile and its
  local harness tests, and reconciled the open-gate indexes. Runtime code,
  deployment settings, and database state were not changed.
- **Evidence:** all read profiles passed with 0% request failures and 100%
  checks; the 50-VU run reached 95.07 requests/s with 22.94 ms p95. The
  Geodata API remained 2/2 Ready, its worker 1/1 Ready, and HPA at 2 replicas
  after the run. No fixture or test data was created.
- **Qualification limits:** the catalogue contained only ten entities, so
  these read results do not establish broad capacity. Independent-row writes,
  large-catalogue performance, a new-image canary, rollback and node/storage
  failure remain open. The live write credentials were unavailable, and the
  single-node stateful failure tests remain unsafe on this host.
- **Harness verification:** 18/18 local Node tests passed for the new
  independent-edits setup, row assignment, bounds, and exact-run cleanup.
  Geodata Python quality (run 108) and image build/publish (run 126) passed.
  No runtime image was deployed because the change only touched load-test
  tooling; the live Deployment remains on its recorded image digest.
- See the [Phase 5 evidence](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md)
  and [scaling roadmap](../geodata/horizontal-scaling-roadmap.md).


## 11 October 2026 — Cross-repository architecture review

- **Prompt used:** “Review the entire architecture. Check the individual
  repositories. Identify essential flaws, duplicate and redundant code and API
  endpoints, unnecessary complexity in all repos and services, and create a
  REVIEW.md document with inconsistencies and a phased mitigation plan, tasks,
  and exit criteria ready for Codex implementation. Identify unfinished phases
  in other work in progress and update the To do and other relevant Docs pages.”
- **Scope:** reviewed current main sources for Identity, Programme, Activity,
  Geodata, Operations, Contracts, Deploy, Platform, Docs, both web clients, and
  the shared .github workflow. Service repositories remain authoritative;
  Deploy and Platform runtime/contract copies are treated as mirrors.
- **Findings:** recorded a duplicate destructive Geodata cleanup path in the
  canonical OpenAPI document, Operations source/runtime mirror drift affecting
  the PostgreSQL sampler, retired NATS paths still advertised by OpenAPI,
  duplicated shared helpers, a large Geodata API module, and unfinished
  Geodata scaling, logging, API consolidation, Programme configuration, and
  metrics-coverage work. Runtime impact of the source/image discrepancy was
  not independently verified.
- **Plan and status at that review point:** added
  [REVIEW.md](../../REVIEW.md) with proposed phases, individual tasks, exit
  criteria, and a self-contained prompt for each phase. Mitigation had not yet
  started at the time of that read-only review. The original review changed no
  runtime code, API, database, or deployed state and ran no tests. The later
  completion of Phases 1–4 is recorded in the next entry.



## 11 October 2026 — Cross-repository review mitigation, Phases 1–4

- **Prompt used:** “Fix REVIEW.md phases 1–4 iteratively. Make sure to retire
  all documentation in this repo and elsewhere for removed API endpoints and
  functionality. Use own criteria and general best practices, don't ask me.
  Document all work done in docs repo and other repositories as well. Leave a
  clean and tidy documentation everywhere.”
- **Phase 1:** added the Contracts-owned source/mirror manifest and CI audit.
  It checks 44 source files, 92 file mirrors, and five exact tree mirrors.
- **Phase 2:** established Platform's canonical logging source and exact
  synchronization; retained service-specific HTTP, persistence, upload,
  authorization, and metrics adapters with their owners.
- **Phase 3:** extracted Geodata route registration into catalogue, import,
  and job groups while keeping state, handlers, transactions, and data
  ownership in Geodata.
- **Phase 4:** removed unused pre-live aliases and their runtime headers,
  registries, and counters. The current contract has 132 operations and 170
  route registrations with no missing or unregistered routes or duplicate
  operation IDs. Audited lifecycle commands remain.
- **Evidence:** source revisions and passing workflow runs, Fleet
  Ready=True at Deploy f7d2e65, Helm revision 246 deployed, and Deployment
  readiness are recorded in the
  [phase evidence report](../architecture/evidence/review-phases-1-4-2026-10-11.md).
  Phase 0 closure is recorded in the following timeline entry.

## 11 October 2026 — Cross-repository review Phase 0 contract safety

- **Prompt used:** “Decide how to resolve: Phase 0 remains open for
  duplicate-YAML-key rejection and the final handler/mirror check.”
- **Decision:** use a standard-library Psych AST validator in Contracts CI.
  Self-test that duplicate keys in one mapping fail and same-named keys in
  separate mappings pass. Merge the two cleanup path definitions into one
  operation that matches the Geodata handler, including the `lt-` identifier
  constraint, both supported global-admin roles, environment and production
  opt-ins, exact confirmation, active-work and linked-data guards, bucket
  restrictions, and all response fields.
- **Changes:** Contracts owns the canonical OpenAPI and validator; its root
  copy and the Platform bootstrap mirror are synchronized byte-for-byte. The
  regenerated route inventory records 131 operations and 170 service route
  registrations. No runtime handler or deployed behavior changed.
- **Evidence:** Contracts commits
  [1ed6efb](https://github.com/myota-platform/myota-contracts/commit/1ed6efb476879da3d10abb20325a104e5f504e24)
  and
  [36cf244](https://github.com/myota-platform/myota-contracts/commit/36cf244bedaf6071e50d7c5aef973ccd890128c4)
  pass [CI run 38157218999](https://github.com/myota-platform/myota-contracts/actions/runs/38157218999).
  Platform mirror commit
  [2426e4d](https://github.com/myota-platform/myota-platform/commit/2426e4da186aca49cfa00a58a83746f3573cdf66)
  passes [CI run 38157179490](https://github.com/myota-platform/myota-platform/actions/runs/38157179490).
  The [Phase 0 evidence record](../architecture/evidence/review-phase-0-2026-10-11.md)
  contains the exact changed paths and results.


## 11 October 2026 — Structured logging rollout, Phases 1–5

- **Prompt used:** “Implement observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- **Phase prompt basis:** the plan's [Phase 1](../observability/prompts/logging-phase-1-shared-foundation.md),
  [Phase 2](../observability/prompts/logging-phase-2-loki-platform.md),
  [Phase 3](../observability/prompts/logging-phase-3-http-tracing.md),
  [Phase 4](../observability/prompts/logging-phase-4-workers-nats.md),
  and [Phase 5](../observability/prompts/logging-phase-5-grafana-correlation.md)
  prompts; Phase 6 and 7 acceptance requirements were reviewed.
- **Follow-up prompts used:** “Continue with the implementation of phase 3,
  decide any blocker based on best criteria and established best practices,
  verify exit conditions.” The latest prompt for this work was: “Implement
  observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- **Implementation and decision:** fixed the four service runner entrypoints
  that bypassed shared logging initialization, published and deployed their
  successful CI image digests, and confirmed all application/worker replicas
  Ready. Geodata boundary, load-harness, and upload-recovery checks passed.
  Loki stores OTLP event strings as the log body and attributes as structured
  metadata; the Grafana JSON parser and JSON-regex trace link did not match this
  representation. Deploy and Platform now query metadata directly and use a
  label-type `trace_id` derived field.
- **Live evidence:** a non-mutating Identity `/healthz` request produced a
  matching Loki log and Tempo `GET /healthz` span; live LogQL dashboard query
  shapes succeeded. No business data was created. Helm revision 225 mounted the corrected datasource/dashboard, and Grafana
  logged successful provisioning. All seven mounted dashboard queries returned
  successfully against live Loki. Fleet later returned to Ready=True on Deploy commit `904bef2`; Helm revision
  227 is deployed after rollback to 226. Authenticated UI links in both
  directions have not yet been verified.
  Phase 6 retention/growth/capacity/privacy and Phase 7 PostgreSQL/PostGIS work
  remain incomplete. See [logging evidence](../observability/evidence/logging-implementation-2026-10-11.md),
  [phase prompts](../observability/prompts/README.md), and
  [implementation plan](../observability/logging-implementation.md).

## 11 October 2026 — NATS migration Phase 6 fact-stream cutover

- **Prompt used:** “Finish implementing phase 6 of
  https://github.com/myota-platform/myota-docs/blob/main/docs/operations/messaging/nats-event-migration-plan.md.
  No metrics, no shadowing, no canary. Usual decision-making and
  documentation requirements and consistency.”
  The copyable, self-contained execution prompt is recorded in the Phase 6
  section of the [migration plan](../operations/messaging/nats-event-migration-plan.md#phase-6).
- **Decision and implementation:** Used one direct gated cutover. The
  Deploy-owned Helm pre-upgrade migration changed the empty, legacy
  Interest-retained `MYOTA_EVENTS` to finite Limits retention with only
  `myota.events.>`. Its one-time migration setting is disabled after cutover;
  topology drift validation remains enabled. All three database-specific relay
  copies now enforce the fact-only stream, while Activity and Geodata work stay
  on separate WorkQueue streams. No runtime service business logic changed.
- **Evidence:** Preflight found zero stream messages/bytes, a drained Activity
  durable and owner outboxes, and preserved six unresolved Geodata DLQ rows.
  The live stream caps are Events 1 GiB/500,000/1 MiB, Activity work
  1 GiB/250,000/1 MiB, and Geodata work 3 GiB/500,000/1 MiB, each with 30-day
  max age, file storage, one replica, and DiscardNew. The disposable K3s test
  verified migration refusal/target validation and idempotent rerun without
  publishing a message; its namespace was deleted.
- **Rollout:** Deploy focused Python quality and Helm validation passed
  (runs 38099669717 and 38099669405). Helm revision 213 is deployed with the
  migration flag off. Fleet is Ready=True at Deploy main commit
  `dfc423d26bd79a82b1177b9e5d51e2d1196580c9`; every MyOTA Deployment has its
  desired replicas ready. No shadowing, canary, new metrics baseline, or
  production test data was used. No Phase 6 database object was obsolete.
- **Limits:** Previously Interest-expired facts remain unrecoverable from NATS.
  PostgreSQL remains authoritative and JetStream is only a bounded transport
  and replay window. Off-node broker recovery is still deferred. See the
  [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md),
  [ADR-0008](../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md),
  and [live topology diagram](../architecture/diagrams/nats-event-migration.md).

## 11 October 2026 — NATS Surveyor retirement implementation

- **Prompt used:** “check implementation of
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/nats-surveyor-migration.md,
  implement missing parts, update documentation, architecture and any other
  repos with dependencies.”
- **Service changes:** Operations removed JetStream inspection while retaining
  storage status and Grafana identity; Admin removed the old page, and Geodata
  removed duplicate broker polling. CI passed after fixing the Operations
  startup/formatting and removing the obsolete browser assertion.
- **Deployment decisions:** Surveyor 0.9.11; ten upstream dashboards pinned at
  `0539d22d7bc7a4574c7f376f1dde37f00bfe2ee9` plus adapted dashboard 16256;
  15-second scrape; existing resource limits; 15-day Prometheus retention;
  JetStream caps 7 GiB file / 1 GiB memory; warning/critical capacity alerts
  at 80/90% for ten minutes. Keep monitoring cluster-internal.
- **Recovery and live checks:** The initial Helm render emitted scientific-
  notation byte caps and NATS rejected the replacement while the old server
  remained healthy. Revision 204 restored the prior release; decimal-string
  limits and `Recreate` were then applied. Helm revision 208 is deployed with
  7 GiB/1 GiB limits and the replacement Operations, Admin, and Geodata images.
  Grafana lists eleven NATS dashboards; a synthetic in-memory stream was
  observed and deleted. Operations migration 003 / core migration 005 removed
  the snapshot table/index after the verified external backup gate; storage
  history remains. Fleet is Ready=True on Deploy main commit
  `d0d2f50846fe2c9e239d46a48809860f4b98d7f5`, with 64/64 resources ready and
  matching applied/desired IDs; Phase 2 is complete. See the
  [monitoring evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md)
  and [roadmap](../observability/nats-surveyor-migration.md).

## 11 October 2026 — Phase 5 early observation closure and retirement

- **Prompt used:** “Ok, complete the following phase 5 item now: Complete the
  24-hour rollback observation after immutable-image Helm revision 193, then
  recheck replacement/legacy filters, backlog counters, migration markers,
  owner-row recovery age and Fleet readiness. Retire only the four legacy
  Geodata durable definitions after this check. The earliest time is
  22:03:29 UTC on 11 October 2026. Ignore the earliest possible time, but
  conduct any additional tests needed to ensure this is safe, cleaning up
  test data afterwards.”
- **Timing boundary:** Fleet was Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a at 22:03:29 UTC on 10 October.
  At the user's explicit instruction, the 24-hour elapsed-time requirement was
  waived and checks closed at approximately 22:13:48 UTC on 10 October,
  about 10 minutes 19 seconds after the anchor. This is not a claim that a
  full 24-hour observation elapsed.
- **Final check and retirement:** Helm revision 193 remained deployed and
  Fleet and all MyOTA Deployments were Ready. Both streams were present and
  empty; the four old Geodata filters were exact with zero pending,
  ack-pending and redelivery; all four replacement durables retained exact
  filters, zero counters and one waiting pull. Activity notification
  remained. Migration 021 columns/indexes were present and the four
  age-bounded recovery queries had zero due rows. Deleted the four exact
  legacy Geodata consumers sequentially, verifying absence after each.
- **Post-check:** The shared event stream, Activity notification durable,
  Geodata work stream and all four target durables remained intact. A
  disposable in-memory broker test verified deletion semantics without
  publishing messages. Both Phase 5 disposable namespaces were deleted and
  verified absent; no production test data or broker messages were created.
- **Test limitation:** Two full delivery-suite attempts each exposed one
  timing-sensitive immediate ACK-counter assertion. Focused ACK verification
  passed and later private-stream counters settled at zero; the full suite is
  not claimed as a clean pass.
- **Status:** Phase 5 is complete within this evidence boundary and the
  explicit user waiver. Phase 6 fact-stream retention work is separate. See
  the [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md)
  and [migration plan](../operations/messaging/nats-event-migration-plan.md).


## 10 October 2026 — NATS migration Phase 5 Geodata cutover and recovery work

- **Prompt used for this phase:**

  ```text
  Continue work implementing phase 5 iteratively,, same criteria and instructions as last phase.
  ```

- Moved all four Geodata work consumers to the file-backed, bounded
  `MYOTA_GEODATA_WORK` WorkQueue and exact pull subjects/durables. Production
  workers use target subjects; the four legacy Geodata durables remain empty
  and inactive during the rollback observation. The Interest-retained
  `MYOTA_EVENTS` stream and Activity notification durable remain live.
- Applied Geodata migration 021 with owner-row dispatch timestamps and recovery
  indexes. Source rows and outbox remain authoritative. No Phase 5 database
  schema, source rows, history or dead letters were removed.
- The partial-deletion recovery fix keeps failed Geodata work retryable after an
  Activity cascade. The Activity handler/repository idempotency fix persists
  the request key and result transactionally. Five rounds of eight concurrent
  same-key requests each produced one cascade outbox fact.
- Isolated host-K3s checks passed: 154 Geodata tests (one optional skip), 40
  Activity tests (one optional broker skip), 25 relay/topology tests, actual
  database refusal/reconnect, same-node NATS PVC restart, two-database
  cascade-failure redelivery, cancellation racing acknowledged preprocessing,
  and the connected expiry → owner-row/outbox reconstruction → completion
  chain. The disposable namespace/PVC, test fixtures, API process and
  port-forwards were removed.
- A production inspection found that image digest settings changed only pod
  annotations while the container refs remained mutable `:latest` tags.
  Deploy now renders all configured first-party runtime, worker, provisioner
  and scheduled-job digests as immutable `repository@sha256:…` refs; Platform
  mirrors those templates. Helm revision 191 deployed at 21:28:41 UTC on
  10 October. Fleet reports Ready=True at Deploy commit `a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109` with
  60/60 resources. Activity, Geodata and shared-runtime pod refs and ImageIDs
  match the configured digests. Activity idempotency fix is live.
- Read-only broker verification found `MYOTA_GEODATA_WORK` file-backed,
  one-replica WorkQueue with finite limits and four exact pull durables; all
  target messages/counters are zero with active pull workers. `MYOTA_EVENTS`
  remains file-backed Interest retention with four inactive, empty legacy
  Geodata durables and its live Activity notification consumer.
- **Previous observation anchor (superseded):** Helm revision 192 completed at 21:39:22 UTC
  on 10 October after a chart release restarted the pods with unchanged image
  digests. Fleet is Ready=True at Deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a`, with 60/60
  resources. The 24-hour rollback observation now ends no earlier than
  21:39:22 UTC on 11 October. Recheck topology, recovery and Fleet, then retire
  only the four old Geodata durables. A read-only 21:52 UTC sample confirmed
  both streams healthy, zero work messages, zero legacy pending/ack-pending/
  redelivery, and waiting replacement workers. This early sample is not the
  24-hour gate; keep all four old durables. No database object is obsolete;
  migration 021 and authoritative records remain. Phase 6 remains separate.
- **Source commits:** Activity idempotency changes in
  `myota-activity-service` are handler `cdea2ba`, repository `d364932`,
  test `6448fc2`; mirrors are `6dbab48`/`9953b00` in Platform and
  `b76f825`/`259e2a7`/`84764e9` in Deploy. Image reference hardening:
  [Deploy `a68eedd`](https://github.com/myota-platform/myota-deploy/commit/a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109)
  and [Platform `a184baa`](https://github.com/myota-platform/myota-platform/commit/a184baac3f36e0272cbc79107f4b362139de7515).
  See [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md).

## 10 October 2026 — NATS migration Phase 3 domain-event consumers

- **Prompt used:** “Continue with the implementation of phase 3, decide any
  blocker based on your best criteria and established best practices, verify
  exit conditions.” This is the single self-contained Phase 3 prompt in the
  [migration plan](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-3).
- **Implementation:** Activity now consumes only the 21 registered Identity
  and Geodata fact subjects through stable durable `activity-notifications-v1`.
  The projection and deduplication/checkpoint commit atomically before ACK;
  diagnostic payloads are redacted; poison records have an audited redrive
  path; and outcome/DLQ metrics have bounded cardinality. Deployment hooks
  provisioned the exact successor before retiring the two broad Activity
  durables. The four Geodata work durables and mixed Interest-retained stream
  were unchanged.
- **Checks:** Activity CI and image publication passed. Deploy Python quality
  and Helm validation passed. Contracts CI initially failed only Ruff
  formatting on its new registry-filter assertion; the formatting fix passed
  the rerun. The 29-test Activity suite passed (one broker test skipped without
  `NATS_TEST_URL`); a separate disposable K3s/PostgreSQL/NATS drill verified
  duplicate/ACK-loss handling, poison redrive, metrics, and clean shutdown.
- **Live result and recovery:** On the local K3s cluster, the exact Activity
  durable is live with zero pending, ack-pending, or redelivered messages; one
  pull is waiting. Five stream durables remain: Activity plus the four
  unchanged Geodata consumers. The first migration hook used a stale pinned
  image digest; an additive idempotent Activity migration was applied by a
  short-lived repair Job, then reapplied successfully by normal migration Job
  `myota-migrations-169`. The repair Job was deleted. Helm revision 170 is
  deployed and Fleet reports 60/60 resources ready, but the Fleet bundle still
  reports `WaitApplied` after recovery. Phase 3's two documented exit criteria
  pass; resolve that Fleet status divergence before Phase 4 runtime changes.
- **Remaining scope:** The mixed `MYOTA_EVENTS` Interest stream, producer paths,
  Activity DB-polled work, and Geodata work paths remain as before. Payload
  privacy/schema enforcement, work-command migration, and final topology
  cutover remain open. See the [Phase 3 evidence](../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
  [consumer runbook](../operations/messaging/activity-notification-consumer.md),
  and [migration plan](../operations/messaging/nats-event-migration-plan.md).

## 10 October 2026 — NATS cluster-internal trust boundary decision

- **Prompt used:** “Remove the requirement for secure NATS / Jetstream because
  communication is cluster-internal. Then continue implementation of phase 1”.
  This is a Phase 1 decision follow-up; the plan retains one copyable Phase 1
  implementation prompt updated to match this decision.
- **Decision:** The workspace owner superseded the earlier proposal to require
  NKey credentials, NATS authentication, and TLS. Keep NATS reachable only via
  the Kubernetes ClusterIP service and trust all pods that can reach it. Do not
  expose port 4222 through an external service, NodePort, LoadBalancer, host
  port, or Ingress. Revisit this if the network boundary or workload trust
  model changes. Operations remains read-only by application behavior; the
  unauthenticated broker does not enforce that distinction.
- **Evidence:** Read-only K3s inspection showed `myota-nats` as ClusterIP
  `10.43.85.18:4222`, no NetworkPolicy in namespace `myota`, and Helm release
  `myota` at revision 157. No NATS deployment, stream, consumer, or application
  runtime was modified for this decision.
- **Phase 1 status:** Removed authentication/TLS and per-role NKey rollout from
  the open work and exit gates. Capacity sizing, payload projection and unknown
  route enforcement, chart readiness/relay compatibility, and off-node restore
  and database reconciliation remain open. See the
  [migration plan](../operations/messaging/nats-event-migration-plan.md),
  [joint review](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  and [topology diagram](../architecture/diagrams/nats-event-migration.md).

## 10 October 2026 — NATS Phase 1 Work checklist follow-up

- **Prompt used:** “OK, complete the section "work" in phase 1 now based on the
  decisions in https://github.com/myota-platform/myota-docs/blob/main/docs/operations/messaging/evidence/phase1-joint-review-2026-10-10.md”.
  This continues the same Phase 1 prompt in the
  [migration plan](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-1);
  no new phase prompt was introduced.
- **Contract checks:** Added fixtures requiring every event to declare a
  consumer disposition and every selected work command to match its registered
  subject, stream, schema, and deploy-owned durable filter. The immutable event
  and work envelopes also require UUID IDs and UTC timestamps ending in Z. The
  Contracts CI workflow passed on commit
  [d57f6b5](https://github.com/myota-platform/myota-contracts/commit/d57f6b5a8f8206639daac0426d08b81d10ba8b0f)
  ([run 38046041494](https://github.com/myota-platform/myota-contracts/actions/runs/38046041494)).
  Three focused contract tests passed, and the five-service source audit
  verified 68 fact types and six legacy work types with no undispositioned
  event-like literals. The UTC work-contract documentation was synchronized in
  [contracts commit 4f6d33d](https://github.com/myota-platform/myota-contracts/commit/4f6d33d65d850f6b238bbc77cabda9be96ed250f)
  and [platform mirror commit 19303f8](https://github.com/myota-platform/myota-platform/commit/19303f8b90e2b64495613837787a5d930a6ee886);
  [Contracts CI](https://github.com/myota-platform/myota-contracts/actions/runs/38046227981)
  and [platform CI](https://github.com/myota-platform/myota-platform/actions/runs/38046230452)
  passed.
- **Recovery documentation:** Added the [JetStream recovery and replay runbook](../operations/messaging/jetstream-recovery.md)
  and linked it from the messaging index, migration plan, and backlog. The
  runbook records PostgreSQL authority, isolated restore/replay, work redrive,
  legacy Interest-retention limits, and remaining qualification requirements.
- **Phase status:** Marked only verified contract, registry, topology-definition,
  provisioning-test, and runbook tasks complete. Authentication/TLS and role ACLs,
  final capacity values, deployment preflight, unknown-route enforcement,
  off-node/PVC-loss recovery, and relay cutover remain open. The review decisions
  are complete; their evidence gates are not.
- **Host verification:** Read-only K3s and Helm inspection found the MyOTA release
  at revision 157 and the node Ready. No production resource or producer/consumer
  path was changed.

## 10 October 2026 — NATS Phase 1 delegated joint review

- **Prompt used:** “OK, condict the Joint review of all payload schemas,
  credentials, production capacity limits, restore/replay qualification, and
  relay-side provisioning on your own bsed on best practices and your best
  criteria and continue phase 1 implementation.” This continues the existing
  [single Phase 1 prompt](../operations/messaging/nats-event-migration-plan.md#chatgpt-prompt--phase-1);
  no new phase prompt was introduced.
- **Schema decisions:** Reviewed all 68 source-derived fact schemas and their
  classifications/dispositions. Accepted them as inventory contracts, with
  producer enforcement gated on minimal projections, prohibited-field/size
  checks, compatibility fixtures, and Geodata preprocessing minimization. The
  ten selected work-command payloads were dispositioned as pending source,
  idempotency, recovery, and size evidence.
  The v1 Geodata preprocessed event still includes internal _records and
  _status; a compact versioned summary is required before enforcement.
- **Security and topology decisions:** Selected per-role NKey credentials from
  operator-managed Kubernetes Secrets, NATS TLS, no shared runtime identity, a
  read-only Operations role, and a separate create-only provisioner identity.
  Relay-side topology mutation will be removed only at a reviewed compatibility
  cutover.
- **Capacity and recovery:** Proposed 1 GiB for facts, 1 GiB for Activity work,
  and 3 GiB for Geodata work, reserving 3 GiB of the existing 8 GiB PVC.
  Evidence queried from deployed relay pods had no pending outbox rows, but
  covered only 2–9 October and included load-test data. Values are not final
  until representative 30-day sizing and pressure/recovery tests pass. Selected
  off-node stream snapshots, isolated restore/replay, declarative consumer
  recreation, and database-authoritative work redrive.
- **Checks and boundaries:** Read-only K3s query showed the node Ready, release
  myota at Helm revision 157, NATS single replica with an 8 GiB PVC, and all
  three outbox databases had no unpublished rows. No messages, streams, consumer
  state, or domain data changed. Credential rollout, off-node/PVC-loss restore, measured sizing, chart
  provisioner readiness, payload enforcement, and relay mutation removal remain
  open. A disposable two-broker NATS 2.10 drill restored three messages and the
  durable ACK position; a fresh durable replayed all three. Temporary namespace,
  CLI, and backup were removed. No production service was modified. Phase 1 is
  not complete. See the
  [joint review evidence](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  [migration plan](../operations/messaging/nats-event-migration-plan.md),
  and [topology diagram](../architecture/diagrams/nats-event-migration.md).

## 9 October 2026 — NATS migration Phase 1 contract and provisioner preparation

- **Contract baseline:** Added a contracts-owned registry for 68 domain facts,
  six proposed Activity work commands, and four proposed Geodata work commands.
  Added event/work envelope schemas and generated per-event outer-envelope
  schemas. Payload contracts remain pending owning-service review.
- **Provisioning:** Added a create-only, drift-checking JetStream provisioner
  that requires explicit finite capacity values and refuses to change existing
  streams or durables. Added a disabled-by-default Compose profile and a
  provisioning runbook. No producer, consumer, or live stream changed.
- **Observed evidence:** Read-only inspection found the deployed single
  `MYOTA_EVENTS` stream still uses Interest retention and has no finite
  byte/message caps; the NATS PVC requests 8 GiB and server args show no auth
  configuration. Three outbox databases had 19,103 rows in the measured 30-day
  query window, but the rows span only 2–9 October and include test load, so
  they do not establish 30-day capacity. The Phase 1 evidence report records
  exact per-database figures and limits.
- **Verification and disposition:** Two contract tests and three topology
  tests passed; Python compilation and diff checks passed. An isolated host-K3s
  JetStream broker created all target streams and work durables, accepted an
  identical second provision, rejected changed capacity configuration without
  mutation, and passed a subsequent matching recheck. The temporary namespace
  was deleted. Live provisioning, restore/replay qualification,
  least-privilege credentials, payload schemas, and relay-side provisioning
  removal remain open. Phase 1 is not complete.
  See the [evidence record](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md),
  [plan](../operations/messaging/nats-event-migration-plan.md), and
  [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
  The contracts #2, deploy #4, platform mirror #1, and organization profile #1
  PRs are merged. Merge records are linked from the evidence report. This closes
  review status only; Phase 1 exit gates remain open.
- **CI follow-up:** Corrected Ruff formatting in the contracts and deploy
  implementations and synchronized platform mirrors. Contract checks, deployment
  Ruff formatting/lint, and platform tests plus Ruff formatting/lint pass. The
  platform image-build job timed out requesting a Docker Hub token on two
  PR-CI attempts, leaving its image status unverified at that point. The Phase 1
  prompt above remains the single prompt for this phase; latest main-branch image
  build and publish results are recorded below. On 10 October, the deploy-owned
  provisioner was hardened to pin and verify consumer replay, pull-wait, delivery,
  storage, and payload settings; four focused topology/provisioner tests passed.
  The next contract step derives payload shapes for all 19 Identity facts from
  `myota-identity-service/identity.py`, keeps additive fields accepted, marks
  data classification, and adds contracts CI assertions. This is source-derived
  evidence pending Identity owner/privacy review; payloads for the four other
  producer services remain open. The existing Phase 1 prompt was used; no new
  phase prompt or runtime migration was introduced. Contract CI passed on
  commit `99547de`; the synchronized platform mirror passed unit, Ruff, and
  container checks on `529631c`.
  A subsequent source pass derived schemas for all 12 Programme facts, including
  content and policy review histories. Untyped producer-controlled configuration/content fields
  remain unconstrained; review metadata is classified. The same Phase 1 prompt
  continues. Identity and Programme joint review and three other service payload
  schemas remain open.
  The Programme contract commit `677f8ae` passed Contracts CI, including schema
  assertions and Ruff; platform mirror commit `b8c8507` passed unit, Ruff, and
  container checks.
  The next source pass derived additive schemas and sensitivity classifications
  for all 10 Activity facts from `myota-activity-service/activity.py`,
  `myota-activity-service/activity_repository.py`, and
  `myota-activity-service/awards.py`. Flexible rule/configuration and nested asset
  shapes remain open; joint owner/privacy review is still required. Contracts
  commit `2b8bddaf` and platform mirror commit `bf6f2727` are on `main`. Local
  contract checks pass; Activity Contracts CI passed in
  [run 38040217352](https://github.com/myota-platform/myota-contracts/actions/runs/38040217352).
  The existing Phase 1 prompt was reused. No producer/consumer runtime path changed.
  Direct-main commits `1076584` (deploy) and `b047a00` (platform) passed Ruff,
  deploy gateway/service image builds, platform tests, container build, and publish.
  See the [deploy checks](https://github.com/myota-platform/myota-deploy/commit/107658467eb708981322649e448832e272daccbb/checks)
  and [platform checks](https://github.com/myota-platform/myota-platform/commit/b047a00e15ecc619e3589fffee37a1aa779ff59a/checks). This follow-up did not change
  a live stream or producer/consumer runtime path. The same Phase 1 prompt
  remained in use for the source audit follow-up below. The contracts main-branch
  source audit passed in CI run 38033202206; the platform mirror tests, Ruff, and
  container check also passed on commit `038cd90`.

**Prompt used** (the single copyable Phase 1 prompt in the plan; retained for the
10 October follow-up audit):

```text
Implement the selected Phase 1 NATS contract and topology decisions recorded in
myota-docs/docs/operations/messaging/nats-event-migration-plan.md and ADR-0008. First read
the Phase 0 inventory and repository ownership map. Do not expand scope beyond the
approved topology.

Review the Phase 0 evidence ownership and gates table. Contract, schema, registry,
and isolated provisioning work may proceed, but do not apply live stream changes
or change producer/consumer behavior until the relevant gap is closed or the
specified owners record the required time-bounded risk acceptance and recovery
plan.

Update the authoritative myota-contracts event documentation/schema and any required
owning-service helpers. Define the versioned envelope, stable message ID, subject
naming/registry, event versus work-command classification, consumer-group naming,
unknown-version behavior, payload bounds, correlation/causation fields, and backward
compatibility rules. Update synchronized myota-platform/myota-deploy contract copies
only through their documented sync process.

Make JetStream stream and durable-consumer provisioning deterministic and
drift-checked. Register every producer subject, but provision durables only for
independent consumer groups with an intended business use; do not create broad
no-op consumers for Limits-retained facts. Provision required consumers before
relying on their processing. Implement the single controlled provisioner specified
in ADR-0008; relay replicas must not concurrently mutate stream configuration.
Validate all correctness-sensitive consumer settings.
Add least-privilege credentials and deployment configuration for each relay/worker
role. Add registry/contract checks that fail when a producer subject lacks schema,
owner, consumer disposition, provisioning and documentation.

Document retention limits, restore/replay and recovery procedures; preserve
PostgreSQL as system of record and Operations as read-only broker inspection. Update
myota-docs and operator docs. Add fo…6439 tokens truncated…e public gateway health check
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


## 10 October 2026 — Geodata payload contract source pass

- Added additive, source-derived schemas and classifications for all 27 Geodata
  facts from `myota-geodata-service/geodata.py`. Registry tests assert the
  source-derived evidence, classifications, and representative payload fields.
  Full entity and deletion-job payloads remain additive because their nested
  records are source-owned and extensible.
- The source review exposed that `geodata.import.preprocessed.v1` currently
  receives the complete preprocessing result, including internal `_records` and
  `_status`; `_records` can contain imported source features. This requires
  payload minimization and Geodata owner/privacy review before producer
  enforcement. No runtime event path changed.
- Local checks passed: contracts tests 2/2, all 68 schema generation, workspace
  source audit, Ruff lint/format, and byte-identical contracts/platform mirror
  comparison. Contracts CI passed for commit `e34de821` in
  [run 38042564323](https://github.com/myota-platform/myota-contracts/actions/runs/38042564323);
  platform mirror CI passed for `63cec55e` in
  [run 38042622987](https://github.com/myota-platform/myota-platform/actions/runs/38042622987).
  Joint owner/privacy review remains open. The Phase 0 audit found no Operations
  event-producing call sites, so Operations payload schemas are not applicable
  unless that service becomes a producer. The existing Phase 1 prompt was
  reused; no new phase prompt or live topology change was introduced. See the
  [evidence record](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [Phase 1 plan](../operations/messaging/nats-event-migration-plan.md).


## 10 October 2026 — Phase 1 contract/topology closeout

- Added bounded per-command payload schemas for all ten selected work types,
  with owning database row IDs and producer/consumer source references. The
  Contracts suite passes 4/4 tests; the workspace source audit found no
  undispositioned Python event-like literals.
- Added the optional fail-closed Helm pre-upgrade provisioner hook and chart
  render checks. It remains disabled until the Phase 2 compatibility gate is
  confirmed. No live relay, producer, consumer, or `myota` namespace resource
  changed.
- Local K3s qualification verified provisioner create/idempotency/drift,
  replacement local PVC restore, bounded replay, and `DiscardNew` capacity
  rejection in `nats-phase1-verify-20261010`; the namespace and its resources
  were deleted. Off-node recovery is deferred at the project owner's direction.
  The short capacity sample and accepted 1/1/3 GiB starting caps are documented
  as bounded single-node decisions, not 30-day measured capacity.
- Deploy tests passed 36 tests with one Pillow-dependent skip. Helm lint passed
  for default and Spainip values; the optional pre-upgrade hook rendered only
  when explicitly enabled and confirmed. Docs, org profile links, and platform
  mirrors were updated.
- Post-push CI passed: [Contracts run 38051378258](https://github.com/myota-platform/myota-contracts/actions/runs/38051378258),
  deploy Python and Helm validation runs 38051333534/38051332975, and
  [platform mirror run 38051342419](https://github.com/myota-platform/myota-platform/actions/runs/38051342419).

**Prompt used for this Phase 1 continuation:**

```text
Address and work on the remaining Phase 1 tasks until completion, iteratively
if needed. If a decision is required, make it yourself based on your best
criteria and best practices.

Don't use the cloud-environment skill. Conduct work on the local K3s server in
a separate namespace and clean it up when done. Off-node recovery is not
required for the current scope.
```

The Phase 1 prompt in the [migration plan](../operations/messaging/nats-event-migration-plan.md)
was used as the implementation basis and updated to reflect these accepted
capacity, trust-boundary, recovery, and production-cutover decisions.

## 10 October 2026 — NATS Phase 2 relay hardening

- **Prompt used:**

  ```text
  OK. Go ahead and implement phase 2 iteratively and until exit criteria are complete.
  Take sane and safe decisions based on your best criteria and best practice without external review.
  ```

- **Relay and contract coverage:** Added generated runtime routing data from the
  canonical Contracts registry. The relay validates registered fact types,
  producer ownership, UUID identity, UTC timestamps, exact subjects, and the
  configured message-size cap. Facts use dotted registered subjects and stable
  `Nats-Msg-Id`; six current Geodata work source event types remain on their
  legacy subjects pending the controlled work-stream phase.
- **Failure and topology behavior:** Publish acknowledgement precedes the
  database mark. A failed mark leaves the row retryable with the same message
  ID; retry delay is capped, and terminal failures can be inspected and
  redriven from their source database with an audit record. Relay startup now
  validates legacy stream/durable settings read-only. The Geodata worker no
  longer removes a retired push durable at startup.
- **Retention and operations:** Added resolution/audit migrations for core,
  Activity, and Geodata dead letters. Geodata import cleanup preserves
  unresolved dead letters. Added per-relay backlog/retry/DLQ metrics, three
  scrape targets, alerts, and a dashboard; Operations remains metadata-only.
- **Verification:** The workspace audit verified 68 registered facts and six
  mapped legacy work types with no undispositioned event-like source literals.
  Contracts tests passed 5/5, deploy relay tests 17/17, and Geodata retention
  tests 6/6. The Activity duplicate-delivery regression passed 2 tests; its
  broker-backed overlap/restart test was skipped because it was not configured
  with an isolated broker. Ruff, Helm lint/render, Compose/Collector/alert/dashboard parsing,
  migration idempotency, live legacy metadata inspection, actual same-ID
  JetStream deduplication, publish/mark crash recovery, metrics, and audited
  redrive passed. The broker/database drill ran in a temporary K3s namespace
  that was removed after testing; no production broker or database object was
  changed during verification.
- **Boundaries:** The current production stream remains Interest-retained and
  mixed. Activity's broad notification durable and four Geodata work durables
  remain in place. Payload schema/prohibited-field enforcement, Geodata
  preprocessing minimization, consumer/work migration, production watermark
  comparison, and controlled cutover remain open. See the
  [Phase 2 evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
  [migration plan](../operations/messaging/nats-event-migration-plan.md), and
  [topology diagrams](../architecture/diagrams/nats-event-migration.md).
- **Rollout correction:**
  - Helm initially rendered the configured 1 MiB limit as `1.048576e+06`; the
    relay expects an integer string. Quoted the deploy-owned value and synced
    the platform mirror. Helm render and CI passed. A temporary live env update
    restored readiness until Fleet applied the quoted chart value.
  - The migrations Job next ran a cached image missing
    `dead_letter_event.resolved_at`. Updated its pull policy, versioned the
    chart as 0.2.14, and synchronized the current platform digest. Migration
    Job 166 then applied `resolved_at` and its unresolved index in all three
    databases.
  - All three relays are 1/1 Ready with healthy database/NATS gauges and zero
    pending rows. Geodata reports six unresolved DLQs, left untouched. Fleet
    is Ready with 59/59 resources, and Helm revision 167 is deployed. See the
    [Phase 2 evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md).

## 11 October 2026 — NATS Surveyor dashboard query verification

- **Prompt used:**

  ```text
  Check the implementation of
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/nats-surveyor-migration.md,
  implement missing parts, and update documentation, architecture, and any
  dependent repositories.
  ```

- Rechecked the live deployment on the host K3s cluster without writing test
  data. Grafana search returned eleven dashboards in folder `NATS`; all
  thirteen adapted dashboard 16256 expressions returned HTTP 200 and one data
  frame through Grafana's Prometheus datasource API, with no query error.
  Surveyor and NATS health were both 1, and no NATS alert was firing.
- Confirmed Deploy and Platform carry the same Surveyor alert rules (Git blob
  `f20ceece6fbf71e517a28008382f02d157c4560d`). No runtime source changes were
  required. The wider Prometheus snapshot had five separate Identity,
  Programme, and Geodata outbox alerts, recorded as follow-up outside this
  monitoring task.
- Recorded the sample values and alert boundary in the
  [cutover evidence](../observability/evidence/2026-10-11-nats-surveyor-cutover.md)
  and updated the [monitoring roadmap](../observability/nats-surveyor-migration.md)
  and [observability architecture](../architecture/diagrams/observability-architecture.md).

## 11 October 2026 — Phase 7 database observability increment

- **Prompt used:** “Implement observability as per
  https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md.
  Usual iterative approach, take decisions autonomously, proceed phase-by-phase,
  check off individual tasks and deliverables (change to checkmarks) and document
  all evidence. Update all relevant documentation.”
- Added the external-secret-backed monitor-role path, bounded PostgreSQL
  receiver configuration, and six-panel database dashboard in Deploy; Platform
  carries synchronized mirrors. Checked the monitor credentials through
  read-only queries to `pg_stat_database` across the three databases.
- The Collector receiver failed authentication with those same Secrets in both
  the live and disposable diagnostic Collector, while `psql` succeeded.
  Disabled the receiver pipeline to stop repeated failures. The deployed
  dashboard file parses, but its database data panels are not accepted.
- All Deployments report Ready. Helm revision 234 is deployed; Helm history labels it a rollback to revision
233. The Fleet BundleDeployment is Ready=True at sync generation 93. All 67
resources are ready, and all Deployments meet their requested replica counts. Phase 7 remains open; see the
  [evidence](../observability/evidence/logging-implementation-2026-10-11.md)
  and [plan](../observability/logging-implementation.md).

## 11 October 2026 — Structured logging, database signals, and deployed acceptance

- **Prompt used:** “https://github.com/myota-platform/myota-docs/blob/main/docs/observability/logging-implementation.md isn't complete. Decide on any gaps based on what you know and best practices. A transition or verification period isn't needed, as this is not yet a live system. Continue to implement the missing bits. No individual feile commits, one commit per task or item completed.”
- **Decision:** retain bounded PostgreSQL metrics through the Operations read-only adapter after the direct Collector receiver failed authentication. Keep PostgreSQL statement logs out of the telemetry pipeline because they may contain SQL literals and user data. Use Psycopg instrumentation with parameter capture disabled and Collector removal of SQL attributes.
- Operations database metrics, Psycopg spans, Loki capacity scrape/panels, and synced Compose/Helm source are deployed. Helm revision 241 is active with 67/67 Fleet resources ready. Live database metrics and SQL-free Tempo span attributes are documented in the [logging evidence](../observability/evidence/logging-implementation-2026-10-11.md).
- The roadmap remains open for authenticated Grafana navigation, request-to-worker correlation, Loki prohibited-field audit, and dynamic Compose runtime parity. No transition period was added; retention configuration was accepted by config and runtime checks without waiting 14 days.
## 11 October 2026 — Stable NATS logging correlation

- **Prompt used:** Continue the implementation of structured logging, decide on remaining gaps using best practices, and avoid transition or verification periods because the system is pre-live.
- Added a stable relay correlation fallback (`correlationId` → `workId` → `eventId`) to the NATS header and bound that value into consumer log context across the five owning services. Payload schemas remain unchanged.
- Service source CI, worker tests, Deploy validation, and Platform tests passed. Updated images are running and all Deployments are ready.
- The Helm release still reports revision 242 as `pending-upgrade` while Fleet reports revision 241 Ready. The originating HTTP correlation remains unavailable for outbox producers that did not persist it; live request-to-worker correlation is not claimed.

## 11 October 2026 — Current system and documentation reconciliation

- **Prompt used:** “Update the myota-docs repository with the current state of
  the system, including updated diagrams. Summarize evidence and task documents,
  and improve and simplify the overall documentation structure.”
- **Scope:** reconciled Docs against GitHub `main` heads for service, contract,
  deployment, platform, client, and organization repositories; inspected current
  MyOTA K3s Helm/Fleet/Deployment readiness read-only; corrected current-state
  status and architecture diagrams; consolidated root and subject navigation.
- **Evidence:** Helm `myota` revision 246 is deployed. Fleet reports Ready,
  Deployed, and Monitored with 67/67 resources at Deploy commit
  `f7d2e65edd5248d70376cf99ccc98c836b7f95e3`; 23/23 MyOTA Deployments were
  Ready. The single-node limitation and outstanding acceptance gates are
  explicit in [Current system state](../architecture/current-state.md).
- **Documentation decisions:** the new current-state page is the single dated
  snapshot; plans own checklists and dated evidence owns test results. Historical
  records remain snapshots. Canonical diagrams and subject indexes point to the
  snapshot. Compatibility pages remain for published links.
- **Runtime impact:** none. Checks were read-only; no tests, test data, or
  cluster changes were made.


## 11 October 2026 — SeaweedFS monitoring retirement

- **Prompt used:** “Re-read `AGENTS.md`, then work on, implement and close all
  issues related to SeaweedFS monitoring retirement.”
- Removed the Operations SeaweedFS sampler/API and Admin monitoring page, retired
  its sampled-history table with a forward-only migration, and retained only the
  application object-storage and readiness paths. Replaced the Grafana dashboard
  with a validated adaptation of dashboard 19173 and synchronized Platform copies.
- Updated the OpenAPI contract and generated clients, Admin legacy bookmark
  redirect, runbooks, repository ownership map, current-state summary, diagrams,
  README indexes and issue cross-links. Deploy remains authoritative for the
  Collector, Grafana provisioning, dashboard, alerts and Helm configuration.
- Evidence, release identifiers, live schema and endpoint checks, and any
  remaining environment limits are recorded in the linked Deploy, Operations,
  Contracts, Admin and Docs issues. Live workload, endpoint, database, and metric checks confirm the retirement paths.
  The Fleet/Helm revision status remains explicitly reported as pending while its
  applied-state metadata catches up; issue notes preserve that discrepancy.

## 11 October 2026 — Geodata TracerProvider warning repair

- **Prompt used:** “Work on and close the issues related to Geodata
  OpenTelemetry TracerProvider warning in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- Confirmed the provider was installed during startup by
  `myota_logging.configure_tracing()` and then installed a second time from
  `otel.telemetry_for()` on HTTP requests. Geodata now reuses one process-wide
  telemetry instance and its startup tracing provider. Setup failures remain
  visible without exposing endpoint values or interrupting request handling.
- Synchronized the Geodata runtime mirror to Deploy/Platform and the
  Platform-owned shared logger to its seven service/deployment mirrors. Focused
  tests, source CI, image builds, and live health/Tempo checks passed. The
  rollout reached Helm revision 257 with Fleet Ready 1/1 and all 23 Deployments
  ready (27/27 replicas). Geodata #7 and Deploy #14 were closed with verified
  acceptance criteria. The complete evidence is in the
  [TracerProvider repair record](../observability/evidence/geodata-tracerprovider-2026-10-11.md).


## 11 October 2026 — NATS migration evidence/readiness issue closeout

- **Prompt used:** “Work on and close all issues related to NATS migration
  evidence and readiness in
  https://github.com/myota-platform/.github/blob/main/profile/README.md.”
- Reviewed the Phase 0–6 evidence, authoritative Contracts registry, service
  dispositions, the three database relay paths, selected Activity/Geodata work
  consumers, and the deployment-owned provisioning and recovery boundary.
- Updated the Contracts registry status and tests for 68 facts; synchronized
  the registry mirror in Platform. The clean-clone registry suite passed 7/7,
  and the workspace event-source audit verified 68 facts and found no
  unclassified event-like literals. The Platform registry test job passed.
- Read-only K3s/NATS inspection confirmed all three disjoint streams and eleven
  expected consumers. Streams were empty, consumers had no pending work, three
  database relays were ready, Helm revision 257 was deployed, Fleet was Ready
  1/1, and all 23 Deployments (27/27 replicas) were ready. The temporary
  read-only NATS CLI pod was removed without publishing messages.
- Closed six source/contract/service/deployment issues with completion
  evidence. Closed Operations #1 as superseded by the current Surveyor and
  Operations retirement design. Refreshed the organization issue index.
- **Accepted limits:** the owner has deferred off-node recovery. Initial finite
  capacity ceilings are accepted for this single-node scope but are not a
  measured 30-day production qualification. No production event traffic is
  inferred from the zero current stream backlog. Six unresolved Geo DLQ
  records remain untouched.
- **CI caveat:** Contract CI run 38171573878 stopped at the mirror audit and
  reported Identity/Programme handler drift plus the Platform registry copy.
  The registry copy was synchronized afterward in Platform commit deb0d0e;
  a fresh direct audit now reports only the Identity/Programme handler drift,
  but the Contracts workflow has not been rerun after that sync. Platform run
  38171688089 passed its test job but failed Ruff formatting for
  `services/identity.py`. These results are recorded without claiming full
  repository CI success.
- Detailed issue mapping and runtime values are in the
  [NATS readiness closeout](../operations/messaging/evidence/nats-readiness-issue-closeout-2026-10-11.md);
  the [current system snapshot](../architecture/current-state.md) carries the
  reconciled deployment state.
