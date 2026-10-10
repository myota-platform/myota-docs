# Phase 4 Activity work migration evidence

**Date:** 10 October 2026  
**Status:** implementation and isolated qualification complete; production
cutover is pending the staged worker drain and image rollout. This is not a
claim that the six job kinds are already JetStream-backed in production.

## Decision and boundaries

Phase 4 moves exactly six accepted Activity job kinds to the disjoint
`MYOTA_ACTIVITY_WORK` WorkQueue stream: `QSO_INGESTION`, `ADIF_IMPORT`,
`AWARD_RECALCULATE`, `AWARD_EVALUATION`, `PDF_RENDER`, and
`STATISTICS_REBUILD`. A work envelope contains only the stable `jobId`; job
payloads remain in `myota_activity.activity_job`, while ADIF/PDF bytes remain
in object storage. The job/resource row remains the authoritative status and
recovery record. Activity's domain facts continue through its existing outbox
path. `NOTIFICATION_SEND` is excluded because notification rows are already
in-app delivered facts and the old job had no external provider side effect.

The current live cluster is deliberately distinguished from the target:

- Read-only production inspection found 154 `NOTIFICATION_SEND` rows, all
  `SUCCEEDED`; no selected Phase 4 job kinds, `RUNNING` jobs, or queued
  Activity notifications were present at inspection time.
- The live `MYOTA_EVENTS` stream is file-backed with Interest retention and
  contains `myota.events.>` and `myota.geodata.>`; it had zero retained
  messages. `MYOTA_ACTIVITY_WORK` and `MYOTA_GEODATA_WORK` do not yet exist.
- The Activity database still had the legacy `activity_job_claim_idx`, no
  `activity_job_status_kind_idx`, and no `lease_token` column. Helm revision
  173 was Ready; Fleet tracked deploy commit `81c2f3217b67a91a104c1b4797cfd9fa4c0e83ca`.
- Therefore production still runs the old Activity database-polled worker.
  No production job, stream, consumer, or database row was changed during
  this evidence pass.

## Per-kind source and behavior inventory

| Work kind / target route | Producer and transaction | Current payload and side effects | Idempotency, completion, retry, and recovery |
|---|---|---|---|
| `QSO_INGESTION` — `activity.qso-ingestion.v1`, `myota.work.activity.qso-ingestion.v1`, `activity-qso-ingestion-v1` | `myota-activity-service/activity.py` accepts up to `MYOTA_MAX_BATCH_QSOS` (default 5,000) and calls `ActivityHandler._queue_job`; `activity_repository.py::_job` commits the job and ID-only outbox command together. | `activity_job.payload` holds the bounded accepted QSO batch. JetStream carries `{jobId}` only. `insert_qso_batch` adds normalized QSOs, derived aggregate changes, and fact outbox rows. | Stable job UUID plus QSO `deduplication_key`; aggregate updates occur only for newly inserted QSO rows. Job reaches `SUCCEEDED` after handler commit; retries use `available_at`; terminal failure is recorded with the job and Activity work-DLQ row. |
| `ADIF_IMPORT` — `activity.adif-import.v1`, `myota.work.activity.adif-import.v1`, `activity-adif-import-v1` | `activity.py::create_adif_import` stores uploaded content in configured object storage. `activity_repository.py::create_import` commits `activity_import`, the job, `activity.adif.queued.v1`, and the work outbox row in one transaction. | Database job payload contains only the import ID; the object store owns file bytes. The worker scans/parses the object, inserts QSOs, updates import status, and creates an in-app result notification. | Stable import/job ID; QSO insert deduplication and notification key `adif:<importId>:completed|failed`. `activity_import.status` is the import result; `activity_job.status` is the handler result. Retryable infrastructure errors are retried; unavailable/invalid content is a completed handler with import status `FAILED`. Existing scheduled ADIF object retention stays a CronJob. |
| `AWARD_RECALCULATE` — `activity.award-recalculate.v1`, `myota.work.activity.award-recalculate.v1`, `activity-award-recalculate-v1` | Created transactionally by QSO ingestion/correction/cascade paths in `activity_repository.py`; also accepted by the admin endpoint in `awards.py::recalculate_award`. | Job row stores programme, subjects, optional award/version/reason. JetStream carries only the job ID. Worker recalculates service-owned aggregates and upserts `activity_award_progress`. | Source requests use stable scoped idempotency keys such as `qso:<qsoId>`, `batch:<activationId>:<deduplicationKey>`, `correction:<correctionId>`, or award/version/subject keys. Progress is unique by award/version/subject; qualification notices have a stable deduplication key. Job status records completion/retry/failure. |
| `AWARD_EVALUATION` — `activity.award-evaluation.v1`, `myota.work.activity.award-evaluation.v1`, `activity-award-evaluation-v1` | `awards.py::create_evaluation_job` verifies caller ownership/admin authority, projects allowed input, then calls `ActivityRepository.enqueue_job`. The job and outbox command share the transaction. | Job payload stores award, subject, and optional trusted facts; JetStream carries only job ID. The handler loads the award and facts, upserts progress, and may create a qualification notice. | Request `Idempotency-Key` where supplied; progress upsert and unique notification key protect redelivery. Job row reports completion. Failed handler attempts use bounded retry and audited database redrive. |
| `PDF_RENDER` — `activity.pdf-render.v1`, `myota.work.activity.pdf-render.v1`, `activity-pdf-render-v1` | Admin-only `awards.py::create_render_job` queues the issuance ID in the job row and outbox transaction. | The issuance and render specification remain in the Activity database; source/background/signature and generated PDF bytes remain in object storage. The worker reuses the issuance renderer and writes the stable artifact object key. | Stable issuance/job ID and deterministic artifact key make re-render overwrite the same owned artifact; issuance persistence is an upsert. Completion is job status after render and persistence; terminal errors are recorded for audited redrive. |
| `STATISTICS_REBUILD` — `activity.statistics-rebuild.v1`, `myota.work.activity.statistics-rebuild.v1`, `activity-statistics-rebuild-v1` | Admin-only `activity.py::rebuild_statistics` queues an optional programme and a date-scoped idempotency key. Job and outbox row commit together. | JetStream carries only job ID; the job row carries the optional programme. Worker rebuilds snapshots from Activity-owned tables. | Rebuild uses the snapshot uniqueness key `(programme, type, scope, period, algorithm_version)` and upserts, so replay replaces the same projection. Job status captures completion/retry/failure. |

All six commands use `envelopeVersion: 1`; the registered `workType` and
subject carry the `.v1` type version. The registry schemas require an exact
UUID `jobId` payload and reject added command fields. The relay retains the
stable job UUID as `Nats-Msg-Id`; audited redrive creates a new outbox event
ID while preserving the original `workId`.

## Implementation and verification

Authoritative Activity code is in `myota-activity-service`; the matching
`myota-deploy/services/` and `myota-platform/services/` copies and migration
snapshots are synchronized mirrors. Contracts are authoritative in
`myota-contracts`; `myota-deploy/services/event_registry.json` and the platform
copy are generated routing catalogs. Deployment ownership is
`myota-deploy/deploy/helm/myota`.

The implementation changes:

- Adds the atomic job/outbox writes, a compatibility reconciler for legacy
  queued rows, per-kind pull durables, explicit ACK after committed job state,
  bounded retry, lease heartbeat and UUID lease fencing, terminal failure
  storage, and audited database-authoritative redrive.
- Validates the pre-provisioned stream durables and binds without creating a
  default consumer. PDF/ADIF and aggregate rebuilds have separate bounded
  concurrency settings. Long work refreshes both the database lease and the
  JetStream ACK timer.
- Adds `migrations/007_activity_jetstream_work.sql`: it refuses the first
  cutover if any selected job is `RUNNING`; backfills queued jobs using their
  stable IDs and `available_at`; corrects delivered in-app notifications and
  removes synthetic `NOTIFICATION_SEND` jobs; drops `activity_job_claim_idx`;
  creates the status/kind/available-time index; and adds lease, work dead-letter,
  and redrive-audit schema. `001_activity_relational.sql` no longer recreates
  the retired claim index on the deployment runner's repeated migration pass.
- Preserves `activity_job`, its API status/history, outbox, import/issuance
  records, and scheduler-owned retention jobs. No domain history table is
  dropped. The old claim index and synthetic notification-job rows were the
  only objects/data removed because they existed solely for database polling
  or a state-only command.

Local checks on 10 October 2026:

- Activity Ruff check and format check passed; 37 Activity tests passed and one
  optional isolated-notification-broker test was skipped because its test URL
  was not configured.
- Contracts tests passed 7/7; the five-service source audit verified 68 facts,
  16 legacy work source types, and no undispositioned event-like Python source
  literals.
- Deploy tests passed 25/25 across relay/outbox/topology suites; Ruff and
  format checks passed; Helm lint passed. Rendered provisioner values were
  quoted integer strings and the Activity worker had two replicas with a
  600-second termination grace period.
- An isolated host-K3s namespace provisioned `MYOTA_ACTIVITY_WORK` and all six
  durables twice with identical settings. A real NATS/PostgreSQL integration
  sent one registered command per durable and confirmed DB-complete duplicate
  delivery was ACKed, ACK-pending returned to zero, and WorkQueue message count
  returned to zero.
- PostgreSQL migration checks verified: a first migration attempt rejects a
  running selected job; queued selected work creates six ID-only outbox rows;
  the synthetic job row is deleted and notification is delivered; the old
  claim index is absent and replacement index exists; a repeated migration
  succeeds while a JetStream job is running. Repository integration verified
  missing-outbox reconciliation is idempotent, stale lease tokens cannot
  heartbeat or complete, terminal job failure and work-DLQ rows commit
  together, and audited redrive restores `QUEUED` and resolves the prior DLQ.
- Namespace `myota-phase4-validation`, its temporary PostgreSQL/NATS pods,
  ConfigMap, Secret, and port forwards were deleted. A follow-up query found
  the namespace absent.

## Remaining production gate

Production rollout is staged to avoid the previous poller claiming work while
the database backfill runs:

1. Provision and validate the disjoint target stream/durables; this does not
   change the mixed legacy `MYOTA_EVENTS` stream or Geodata queues.
2. Scale the Activity DB-polling worker Deployment to zero and verify the old
   worker pods are gone. Activity APIs remain available and continue storing
   accepted job rows/outbox rows.
3. Run the guarded Activity schema/backfill migration while no old worker can
   claim. Verify no selected `RUNNING` job remains and all queued selected jobs
   have a matching outbox row.
4. Roll out the new Activity API/worker images. The startup/periodic legacy
   outbox repair covers any old API pod that wrote a queued job during the
   rolling update. Verify every selected job has an outbox event and no
   database claim loop remains.
5. After old API pods are gone and the repair reports zero rows, turn off the
   temporary legacy repair loop in a follow-up cleanup; retain the work-row
   source of truth, durable status endpoint, metrics, and audited redrive.

At the production read-only inspection, all six selected work kinds had zero
jobs, so the backfill had no existing selected backlog. Actual production
provisioning, migration, rollout, durable pending/ACK state, metrics scrape,
HPA/termination behavior, and removal of the compatibility repair loop are not
yet verified. Phase 4 exit criteria remain open until those checks pass.
