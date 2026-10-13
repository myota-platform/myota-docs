# Database migration baselines and data-preserving rebuilds

**Status verified:** 13 October 2026 09:45 UTC  
**Scope:** Deploy, Activity, Geodata, Platform, and the local non-production K3s
cluster. This record covers baseline source and disposable runner evidence. It
does not authorize or report a live database rebuild.

## Ownership and synchronized copies

| Database | Schema owner and authoritative baseline | Deployment runner and mirror | Status |
|---|---|---|---|
| `myota_core` | Deploy: [`db/migrations/core/001_core_baseline.sql`](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/core/001_core_baseline.sql); shared by Identity and Programme | Deploy owns [`db/migrations/run.sh`](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/run.sh). Platform mirrors runner and SQL at [`db/migrations/core/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/core). | Fresh and repeat runner verified in disposable K3s DB; current Helm image is still on historical migration chain. |
| `myota_activity` | Activity: [`migrations/001_activity_baseline.sql`](https://github.com/myota-platform/myota-activity-service/blob/testing/migrations/001_activity_baseline.sql) | Deploy mirrors at [`db/migrations/activity/`](https://github.com/myota-platform/myota-deploy/tree/testing/db/migrations/activity); Platform mirrors at [`db/migrations/activity/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/activity). | Fresh and repeat runner verified in disposable K3s DB; current Helm image is still on historical migration chain. |
| `myota_geo` | Geodata: [`migrations/001_geodata_baseline.sql`](https://github.com/myota-platform/myota-geodata-service/blob/testing/migrations/001_geodata_baseline.sql) | Deploy mirrors at [`db/migrations/geo/`](https://github.com/myota-platform/myota-deploy/tree/testing/db/migrations/geo); Platform mirrors at [`db/migrations/geo/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/geo). | Fresh and repeat runner verified against PostGIS 3.4.3 in disposable K3s DB; current Helm image is still on historical migration chain. |

The owner repositories define schema content. Deploy owns the combined database
migration runner and deployment wiring. Platform is a synchronized mirror and
must not become an independent schema authority.

## Baseline and runner evidence

The three baseline source PRs and mirror PRs are merged to `testing`:

- Core/runner: [Deploy PR #18](https://github.com/myota-platform/myota-deploy/pull/18)
  and [Platform PR #3](https://github.com/myota-platform/myota-platform/pull/3).
- Activity: [Activity PR #7](https://github.com/myota-platform/myota-activity-service/pull/7).
- Geodata: [Geodata PR #11](https://github.com/myota-platform/myota-geodata-service/pull/11).

An end-to-end runner rehearsal exposed PostgreSQL psql compatibility failure:
the 16.4 client rejected pg_dump's `\\restrict` and `\\unrestrict`
metacommands embedded in the Core and Activity baseline files. Those lines were
removed from the authoritative/mirror SQL and the compatibility fix was merged
to `testing` in [Deploy PR #19](https://github.com/myota-platform/myota-deploy/pull/19),
[Activity PR #8](https://github.com/myota-platform/myota-activity-service/pull/8),
and [Platform PR #4](https://github.com/myota-platform/myota-platform/pull/4).
The SQL bodies remain byte-identical across their owners and mirrors.

The fixed runner and migration SQL were tested in the isolated
`codex-db-rebuild` namespace using the PostgreSQL 16/PostGIS test pods:

- Fresh install created exactly one `baseline-v1` ledger row per database and
  set all three readiness rows to `ready=true`.
- A repeat run skipped all recorded baselines, retained one ledger row each,
  and remained ready.
- Failure injection hid Core's `public.account` sentinel. The runner failed
  with the expected error and left all three readiness rows false.
- Restoring the sentinel and expected baseline checksum let the runner
  complete and restore readiness across all three databases.
- Runner jobs, ConfigMap, and the three runner-test databases were deleted after
  evidence capture. No live MyOTA database was changed by these checks.

CI evidence for the baseline changes includes Activity run
[38368130423](https://github.com/myota-platform/myota-activity-service/actions/runs/38368130423),
Geodata runs
[38370613580](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370613580)
and
[38370614459](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370614459),
Deploy runs
[38370753170](https://github.com/myota-platform/myota-deploy/actions/runs/38370753170)
and
[38370752130](https://github.com/myota-platform/myota-deploy/actions/runs/38370752130),
Platform run
[38370757925](https://github.com/myota-platform/myota-platform/actions/runs/38370757925).
The compatibility-fix PR runs also passed: Deploy
[38374441554](https://github.com/myota-platform/myota-deploy/actions/runs/38374441554),
Activity
[38374446912](https://github.com/myota-platform/myota-activity-service/actions/runs/38374446912),
and Platform
[38374443683](https://github.com/myota-platform/myota-platform/actions/runs/38374443683).

These checks establish baseline SQL and runner behavior in a disposable
environment. They do not establish Identity/Programme or Activity API/worker
integration acceptance, deployment of the fixed migration image, or safe live
cutover.

## Live runtime boundary

Helm release `myota` is revision 282, created as a rollback to revision 281.
MyOTA Fleet still tracks Deploy `main` commit `9d80741`. The completed live
migration Job `myota-migrations-281` executed the historical numbered chain,
including Geodata migration 022, and completed successfully. It did not execute
the new baseline runner from `testing`.

All live databases remain intact and their readiness markers remain healthy.
No go/no-go has been recorded. Do not drop or recreate a live database until
the fixed runner is built and deployed, baseline issue acceptance is complete,
writers and asynchronous work are quiesced, and a final checksummed archive has
passed isolated restore validation.

## Recovery archive evidence

The following custom-format archives were captured while services were still
active. Each passed archive inspection and an isolated restore. They are
**preliminary rehearsal archives only**, not cutover backups:

| Database | Size | SHA-256 | Isolated restore evidence |
|---|---:|---|---|
| Core | 112,415 bytes | `420822e5185845c98b42ba226e0eeac6917db23c70d091eb77a7415e94c41c64` | 29 public tables restored. |
| Activity | 863,447 bytes | `dc1901822aa198e024e0a1f737e6a89845188df1c6a5ceb9e4b688559eeb4ac` | 26 public tables restored. |
| Geodata | 9,647,821 bytes | `89a321ff6ae221ba4ae1cda3f3cc3e221b3ff9840ea4a9a3153a52b438642034` | 28 public tables restored; PostGIS available; normalized schema matched the fresh baseline. |

Because capture happened before quiescence, these archives do not satisfy the
cutover gate. New archives must be taken after service writers, scheduled work,
outbox relays, and JetStream consumers are paused or fenced and in-flight work
is accounted for. Restore only data into the verified schema; exclude baseline
bookkeeping (`myota_schema_migration` and
`myota_deployment_schema_state`). For Geodata, also exclude
`geodata_schema_feature` so the baseline-owned `row_authority_v1` marker
survives. Preserve sequences, large objects, domain rows, outbox state,
idempotency, and consumer checkpoints.

The rehearsal namespace retains its restore pods and archived databases until
the paired recovery evidence is complete. Remove it after evidence capture and
remove the restricted local archive directory after acceptance.

## Historical migration and object ownership map

The pre-baseline file inventories below were read from the service/deployment
`main` histories. Their schema effects are now consolidated in the single
numbered baseline shown above. The baseline is schema-only: it does not replay
migration-time data copies or seed domain rows.

### Core history and target ownership

Historical Deploy source files under
[`db/migrations/core/`](https://github.com/myota-platform/myota-deploy/tree/main/db/migrations/core):

| Historical migration | Schema effect and disposition in the baseline |
|---|---|
| `001_core.sql` | Initial shared core schema. The current target has Identity account/auth/callsign/security objects; Programme policy, award, jurisdiction and approver-scope objects; and shared outbox, idempotency, consumer and dead-letter infrastructure. |
| `002_operations.sql` | Created `operations_jetstream_snapshot` and its history index. These Operations monitoring objects are retired and absent from the Core baseline. |
| `003_storage_snapshots.sql` | Created `operations_storage_snapshot` and its history index. These sampled SeaweedFS monitoring objects are retired and absent from the Core baseline. |
| `004_outbox_dead_letter_redrive.sql` | Added operator-audited outbox redrive state; retained as `outbox_redrive_audit`. |
| `005_retire_jetstream_snapshots.sql` | Dropped `operations_jetstream_snapshot`; NATS monitoring is owned by Surveyor. |
| `005_retire_operations.sql` | Dropped the Operations sampled-history tables. This historical chain has a second migration also numbered 005. |
| `006_outbox_correlation.sql` | Added nullable correlation to `outbox_event`; retained in the current event infrastructure. |
| `006_retire_storage_snapshots.sql` | Dropped `operations_storage_snapshot`; this history has a second migration also numbered 006. |

The current Core baseline contains 29 public tables:
`account`, `account_credential`, `account_primary_callsign`,
`account_recovery_token`, `account_role`, `activation`,
`approver_scope`, `audit_event`, `auth_revocation`, `auth_session`,
`award`, `award_rule`, `callsign`, `callsign_evidence`,
`consumer_checkpoint`, `consumer_processed_event`, `dead_letter_event`,
`idempotency_record`, `jurisdiction`, `oidc_provider_mapping`,
`outbox_event`, `outbox_redrive_audit`, `programme`,
`programme_policy_version`, `programme_rule`, `qso`,
`security_event`, `service_state`, and `myota_schema_migration`.

Current Identity and Programme persistence is visible in each service's
`common.py`: `service_state`, `idempotency_record`, and
`outbox_event` are selected or written there. Identity's `identity.py` and
Programme's `programmes.py` handle the matching domain objects. The
normalized `account_*`, `auth_*`, `callsign_*`, award, and programme-policy
relations are present in the current catalog, but the inspected API handlers
do not issue SQL against all of them. Preserve their rows through rebuild; the
exact object-to-handler owner mapping remains an acceptance gate rather than
inferring activity from names.

Core `activation` and `qso` are the pre-split Activity-shaped relations.
Current Activity persistence uses `activity_activation` and `activity_qso`
in `myota_activity` through
`myota-activity-service/activity_repository.py`. No current Identity or
Programme SQL path targets Core `activation` or `qso`. Preserve these legacy
Core rows during this rebuild; do not use them as current Activity write
targets.

The operations tables above are specifically absent from
[`001_core_baseline.sql`](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/core/001_core_baseline.sql).
No live Core data was removed in the baseline task.

### Activity history and target ownership

Historical authoritative Activity files under
[`myota-activity-service/migrations/`](https://github.com/myota-platform/myota-activity-service/tree/main/migrations):

| Historical migration | Schema effect consolidated in `001_activity_baseline.sql` |
|---|---|
| `001_activity_relational.sql` | Normalized Activity entities and service-local outbox/idempotency/consumer infrastructure. |
| `002_activity_entity_deletion.sql` | Bounded entity-impact and QSO-cascade indexes. |
| `003_adif_source_retention.sql` | ADIF source-object retention marker/index. |
| `004_adif_failed_source_retention.sql` | Extends the retention index to failed imports. |
| `005_outbox_dead_letter_redrive.sql` | Audited, operator-authorized redrive state. |
| `006_notification_consumer_replay.sql` | Notification replay coordinates. |
| `007_activity_jetstream_work.sql` | Transactional work outbox support; retains API-visible `activity_job`, removes the poll-only claim index and synthetic state-only jobs. |
| `008_outbox_correlation.sql` | Nullable outbox request correlation without changing event payloads. |

The Activity baseline contains 26 public tables:
`activity_activation`, `activity_asset`, `activity_award_definition`,
`activity_award_issuance`, `activity_award_progress`,
`activity_award_request`, `activity_import`, `activity_job`,
`activity_notification`, `activity_notification_redrive`,
`activity_qso`, `activity_qso_correction`,
`activity_statistic_snapshot`, `activity_subject_aggregate`,
`activity_subject_callsign`, `activity_subject_entity`,
`activity_work_dead_letter`, `activity_work_redrive_audit`,
`consumer_checkpoint`, `consumer_processed_event`, `dead_letter_event`,
`idempotency_record`, `outbox_event`, `outbox_redrive_audit`,
`service_state`, and `myota_schema_migration`.

Activity owns the `activity_*` tables. SQL call sites are concentrated in
`activity_repository.py`, `activity_worker.py`, `awards.py`, and
`event_consumer.py`; generic state, idempotency, and outbox support lives in
`common.py`. Activity's README states durable mode does not use generic
`service_state` as its source of truth; retain that compatibility relation
through this data-preserving rebuild. Deploy's
[`db/migrations/activity/`](https://github.com/myota-platform/myota-deploy/tree/testing/db/migrations/activity)
and Platform's matching directory are synchronized copies, not schema owners.

### Geodata history and target ownership

Historical authoritative Geodata files under
[`myota-geodata-service/migrations/`](https://github.com/myota-platform/myota-geodata-service/tree/main/migrations):

| Historical migration | Schema effect consolidated in `001_geodata_baseline.sql` |
|---|---|
| `001_geodata.sql` | Initial PostGIS catalogue, local state, idempotency and outbox infrastructure. |
| `002_qgis_views.sql` | QGIS views and access grants. |
| `003_production_pipeline.sql` | Production import/review pipeline metadata and GIS staging. |
| `004_location_enrichment.sql` | Reverse-geocoded location metadata. |
| `005_location_manual_precedence.sql` | Manual municipality alias and override precedence. |
| `006_unscoped_imports.sql` | Platform-wide imports without requiring programme assignment. |
| `007_relational_entity_persistence.sql` | Relational entity persistence alongside the legacy snapshot. |
| `008_entity_category_assignments.sql` | Relational many-category assignments. |
| `009_candidate_lifecycle.sql` | Unified candidate lifecycle. |
| `010_import_preprocessing.sql` | Staged, reviewable import candidates. |
| `011_import_recovery.sql` | Restart-safe processing leases and source recovery. |
| `012_import_finalization.sql` | Import finalization metadata and staged-row cleanup support. |
| `013_import_retention.sql` | Bounded retention eligibility for finalized imports. |
| `014_resumable_uploads.sql` | S3 multipart upload sessions and parts. |
| `015_jetstream_worker_dispatch.sql` | Durable import/recovery dispatch and legacy repair support. |
| `016_relational_authority.sql` | Writer fence and relational source of truth; preserves legacy snapshot only as archive and does not merge it over authoritative rows. |
| `017_import_cancellation.sql` | Durable cancellation for queued uploads and preprocessing. |
| `018_import_lookup_indexes.sql` | Import-scoped and entity lookup indexes. |
| `019_maidenhead_locators.sql` | Derived Maidenhead locator cells. |
| `020_outbox_dead_letter_redrive.sql` | Audited outbox dead-letter redrive. |
| `021_jetstream_work_recovery.sql` | Bounded dispatch repair marker for lost/expired work. |
| `022_entity_website_description.sql` | Optional public Website and Description entity fields. |
| `022_outbox_correlation.sql` | Nullable outbox correlation; historical path has a second file numbered 022. |

The Geodata baseline contains 27 public tables:
`conflation_candidate`, `consumer_checkpoint`,
`consumer_processed_event`, `dead_letter_event`, `entity_review`,
`entity_type`, `geodata_audit_event`, `geodata_control_record`,
`geodata_edit_staging`, `geodata_entity`,
`geodata_entity_attachment`, `geodata_entity_category`,
`geodata_import_candidate`, `geodata_import_processing_queue`,
`geodata_schema_feature`, `geodata_upload_part`,
`geodata_upload_session`, `idempotency_record`, `import_run`,
`import_schedule`, `outbox_event`, `outbox_redrive_audit`,
`source_reference`, `service_state`, `source_snapshot_manifest`,
`source_snapshot_record`, and `myota_schema_migration`.

Relational entities, categories, controls, idempotency and outbox writes are
implemented in `relational_state.py`; import-run and snapshot lifecycle
handling also uses `geodata.py`, `geodata_import_worker.py`,
`geodata_store.py`, and `import_retention.py`. The service README and
migration comments classify `service_state` as an archive; the import
retention worker removes `source_snapshot_manifest` by import run, with its
records removed by the foreign-key cascade. These are active import
provenance/cleanup relations, not retired monitoring state.

### Retained-object ownership evidence

The table names alone do not prove active ownership. Current source inspection
has established these dispositions:

- Identity and Programme `common.py` each read and write their own row in Core
  `service_state`. This is active JSON-backed service state and must remain
  available through the rebuild.
- Current Activity domain state is stored in `activity_activation` and
  `activity_qso` in `myota_activity`; the generic Activity
  `service_state` table is retained in the baseline for compatibility, while
  durable mode does not use it as the source of truth.
- Geodata `service_state` is a legacy JSON snapshot archive. The Geodata
  migration guide, baseline table comment, and relational-authority tests say
  the live relational runtime does not read or write it. Preserve its existing
  row through the rebuild until a separate disposition is recorded.
- Geodata `source_snapshot_manifest` and `source_snapshot_record` are
  active import provenance/cleanup state. In particular,
  `import_retention.py` deletes manifests by `import_run_id`; these tables
  must be retained and restored with their import rows.
- Core `activation` and `qso` are the pre-split Activity-shaped relations.
  Current Activity uses its prefixed tables in `myota_activity`; no current
  Identity or Programme callsite was found in the inspected persistence paths.
  Keep Core rows through this rebuild for data preservation, and do not treat
  these relations as current Activity write targets.

The exhaustive table-to-callsite mapping is still incomplete for the remaining
Core auth/Programme tables and indexes, functions, views, sequences, triggers,
roles, and grants. The exact evidence to close that gate is a table/object to
source path map validated against the deployed catalog. Do not remove any
retained relation during this baseline/rebuild work. Historical Operations
snapshot objects remain the verified exception: both are absent from the Core
baseline after retirement.


### Current catalog owner and callsite map

The baseline inventory is compared with the deployed PostgreSQL catalogs. These
source maps identify current persistence paths and distinguish them from schema
objects that remain for compatibility. No relation is removed based only on a
missing SQL reference.

#### Core (29 public tables)

| Objects | Current owner and exact source path | Current use / rebuild disposition |
|---|---|---|
| `service_state`, `idempotency_record`, `outbox_event` | Identity and Programme persistence adapter: [Identity common.py](https://github.com/myota-platform/myota-identity-service/blob/main/common.py#L329) and [Programme common.py](https://github.com/myota-platform/myota-programme-service/blob/main/common.py#L329). | Active durable state, idempotency, and transactional outbox. Retain data and constraints. |
| `account`, `account_credential`, `account_primary_callsign`, `account_recovery_token`, `account_role`, `auth_revocation`, `auth_session`, `callsign`, `callsign_evidence`, `oidc_provider_mapping`, `security_event`, `audit_event` | Identity API state is handled by [identity.py](https://github.com/myota-platform/myota-identity-service/blob/main/identity.py) and serialized by [common.py](https://github.com/myota-platform/myota-identity-service/blob/main/common.py#L329). | Current API state is JSON-backed in Identity's `service_state` row. The inspected handler paths do not issue direct SQL against these normalized relations; classify them as retained compatibility schema and preserve any rows. |
| `programme`, `award`, `award_rule`, `approver_scope`, `jurisdiction`, `programme_policy_version`, `programme_rule` | Programme API state is handled by [programmes.py](https://github.com/myota-platform/myota-programme-service/blob/main/programmes.py) and serialized by [common.py](https://github.com/myota-platform/myota-programme-service/blob/main/common.py#L329). | Current API state is JSON-backed in Programme's `service_state` row. The inspected handler paths do not issue direct SQL against these normalized relations; classify them as retained compatibility schema and preserve any rows. |
| `activation`, `qso` | Current Activity persistence is owned by [activity_repository.py](https://github.com/myota-platform/myota-activity-service/blob/testing/activity_repository.py). | These are pre-split Core relations. Current Activity writes `activity_activation` and `activity_qso` in `myota_activity`; retain existing Core rows without treating them as current Activity storage. |
| `dead_letter_event`, `outbox_redrive_audit` | Deploy outbox handling: [outbox_worker.py](https://github.com/myota-platform/myota-deploy/blob/testing/services/outbox_worker.py) and [outbox_admin.py](https://github.com/myota-platform/myota-deploy/blob/testing/services/outbox_admin.py). | Active outbox delivery, dead-letter, and audited redrive state. Preserve for recovery and duplicate-effect protection. |
| `consumer_checkpoint`, `consumer_processed_event` | No active Core consumer SQL callsite was found in Identity, Programme, or the Deploy Core relay paths inspected. Current JetStream consumers use service-local processing state. | Retained compatibility state. Preserve rows; do not infer retirement from the absence of a Core reader. |
| `myota_schema_migration` | Deploy migration runner: [run.sh](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/run.sh). | Active runner ledger. Retain and validate baseline marker/checksum behavior. |

#### Geodata (27 public tables)

| Objects | Current owner and exact source path | Current use / rebuild disposition |
|---|---|---|
| `entity_type`, `geodata_entity`, `geodata_entity_category`, `source_reference`, `geodata_entity_attachment` | Relational entity writes and projection: [geodata_store.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_store.py) and [relational_state.py](https://github.com/myota-platform/myota-geodata-service/blob/main/relational_state.py). | Current authoritative entities, category assignments, source provenance, and attachments. Preserve relations, foreign keys, PostGIS geometry, and indexes. |
| `conflation_candidate`, `entity_review`, `geodata_edit_staging`, `geodata_audit_event`, `geodata_control_record`, `geodata_schema_feature` | Review, control, feature-check, and audit paths in [geodata_store.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_store.py), [relational_state.py](https://github.com/myota-platform/myota-geodata-service/blob/main/relational_state.py), and [geodata.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata.py). | Active review/edit and relational control state; preserve through data-only restore. The schema feature relation gates compatibility checks. |
| `import_run`, `geodata_import_candidate`, `geodata_import_processing_queue`, `import_schedule` | Import lifecycle and asynchronous work: [geodata_store.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_store.py), [relational_state.py](https://github.com/myota-platform/myota-geodata-service/blob/main/relational_state.py), and [geodata_import_worker.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_import_worker.py). | Active import, candidate, queue, and schedule state. Preserve pending/recoverable work and its IDs. |
| `geodata_upload_session`, `geodata_upload_part` | Upload and expiry handling: [geodata.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata.py) and [geodata_retention_worker.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_retention_worker.py). | Active resumable upload/recovery state. Preserve session/part linkage and expiry status. |
| `source_snapshot_manifest`, `source_snapshot_record` | Import provenance and cleanup: [import_retention.py](https://github.com/myota-platform/myota-geodata-service/blob/main/import_retention.py); manifest deletion is scoped by import run and records cascade through the FK. | Active import provenance/retention data. Preserve with the matching import rows. |
| `service_state` | [Geodata README](https://github.com/myota-platform/myota-geodata-service/blob/main/README.md) and [relational_state.py](https://github.com/myota-platform/myota-geodata-service/blob/main/relational_state.py). | Legacy JSON snapshot archive; current relational runtime does not use it as authoritative state. Preserve existing contents pending a separate disposition. |
| `idempotency_record`, `outbox_event`, `dead_letter_event`, `outbox_redrive_audit`, `consumer_checkpoint`, `consumer_processed_event` | Service persistence adapter: [common.py](https://github.com/myota-platform/myota-geodata-service/blob/main/common.py); outbox delivery and recovery: [outbox_worker.py](https://github.com/myota-platform/myota-deploy/blob/testing/services/outbox_worker.py) and [outbox_admin.py](https://github.com/myota-platform/myota-deploy/blob/testing/services/outbox_admin.py). | Active request idempotency, publication, delivery/recovery state. Preserve IDs/checkpoints and pending/dead-letter linkage. |
| `myota_schema_migration` | Deploy migration runner: [run.sh](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/run.sh). | Active runner ledger. Retain and validate baseline marker/checksum behavior. |

The retained-object map is source-based, not a proposal to remove schema. The
catalog comparison and focused service tests remain separate acceptance gates.

### Disposable catalog and integration checks — 2026-10-13

The following focused checks ran only against isolated databases in the
`codex-db-rebuild` namespace. Temporary Core and Activity databases, synthetic
rows, copied SQL, and local dumps were removed after verification. The retained
`restore-core`, `restore-activity`, and `restore-geo` rehearsal databases
were left intact for the open rebuild issues.

| Database | Catalog evidence | Service-path evidence |
|---|---|---|
| Core | The baseline catalog was compared with the archived Core schema. All application tables, constraints, and deployment readiness objects matched; the only additional object in the new target is `myota_schema_migration`, the intended baseline/checksum ledger. The deployment readiness table remains present. | Identity account creation persisted and reloaded from `service_state`, with one transactional `identity.account.created.v1` outbox row. Programme creation persisted and reloaded from `service_state`, with one `programme.created.v1` outbox row. |
| Activity | The baseline catalog compared with the archived Activity schema; the only additional object in the target is `myota_schema_migration`. | Activity API handler create/read passed. A QSO-ingestion job was enqueued, claimed under a worker lease, completed, and read back as succeeded; activation and work-command outbox rows were verified. |
| Geodata | The restored archive and fresh baseline public schemas matched after normalization. The baseline runner verified PostGIS 3.4.3 and spatial indexes. | Geodata CI run [38370613580](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370613580) passed the disposable database migration, two independent API instances and regression suite, worker ACK/recovery tests, and upload-session API/restart-recovery integration. Quality run [38370614459](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370614459) passed. |

These checks close the baseline catalog/service-path gates. They do not deploy
the new runner to the live Helm release or authorize a database cutover; those
are gates in the paired rebuild issues.

## Issue status and remaining gates

The three baseline issues are complete and closed; all three rebuild issues remain open:

| Owner | Baseline issue progress | Rebuild issue progress |
|---|---|---|
| Deploy/Core | [#15](https://github.com/myota-platform/myota-deploy/issues/15) — 7/7 checked; closed | [#16](https://github.com/myota-platform/myota-deploy/issues/16) — 1/7 checked; open |
| Activity | [#4](https://github.com/myota-platform/myota-activity-service/issues/4) — 6/6 checked; closed | [#5](https://github.com/myota-platform/myota-activity-service/issues/5) — 1/7 checked; open |
| Geodata | [#8](https://github.com/myota-platform/myota-geodata-service/issues/8) — 5/7 checked; open | [#9](https://github.com/myota-platform/myota-geodata-service/issues/9) — 1/7 checked; open |

Verified baseline checkboxes and evidence are recorded in those issues. Each
rebuild issue now checks off the checksummed preliminary archive and isolated
restore rehearsal. Those archives predate service quiescence and writer fencing;
they are not cutover backups. Remaining baseline gates are exact historical
migration/object ownership maps and focused
service integration checks (Identity/Programme for Core; API, worker, outbox,
import/recovery paths for Activity and Geodata). The runner is not yet deployed
from the fixed source in Helm.

Rebuild gates remain: deploy the verified runner image; quiesce writers,
workers, scheduled jobs, relays, and consumers; take and rehearse each final
post-quiescence backup; record per-database go/no-go; recreate one database at a
time from its baseline; data-only restore; validate counts, sequences,
extensions, constraints, read/write paths, outbox/idempotency/checkpoints,
readiness, and recovery; then resume services and clean temporary artifacts.
