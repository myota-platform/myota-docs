# Activity JetStream work queues

**Status:** deployed in production through the Phase 4 cutover. Two Activity
workers consume the six exact durables; the database poller and transitional
repair are retired. The production selected-work backlog was zero at cutover,
so production handler execution was not observed. Isolated PostgreSQL/JetStream
qualification processed each work type. See the [Phase 4 evidence](evidence/phase4-activity-work-2026-10-10.md)
and migration [plan](nats-event-migration-plan.md).

## Ownership and command routes

`myota-activity-service` owns `activity_job`, accepted job payloads, business
state, job status, retry eligibility, dead-letter records, and redrive audit.
The Activity outbox writes a small `{ "jobId": "<uuid>" }` payload in the same
transaction as the accepted job row. The relay owned by `myota-deploy` sends it
to `MYOTA_ACTIVITY_WORK` using the registered work type and stable message ID.
ADIF source files and rendered certificate PDFs remain in the configured object
store; NATS carries no file or row batch content.

| Work type | Subject | Durable | Ack wait | Max pending |
|---|---|---|---:|---:|
| QSO ingestion | `myota.work.activity.qso-ingestion.v1` | `activity-qso-ingestion-v1` | 120 s | 4 |
| ADIF import | `myota.work.activity.adif-import.v1` | `activity-adif-import-v1` | 300 s | 1 |
| Award recalculation | `myota.work.activity.award-recalculate.v1` | `activity-award-recalculate-v1` | 120 s | 2 |
| Award evaluation | `myota.work.activity.award-evaluation.v1` | `activity-award-evaluation-v1` | 120 s | 2 |
| PDF render | `myota.work.activity.pdf-render.v1` | `activity-pdf-render-v1` | 300 s | 1 |
| Statistics rebuild | `myota.work.activity.statistics-rebuild.v1` | `activity-statistics-rebuild-v1` | 300 s | 1 |

All durables use explicit ACK, pull delivery, max delivery 8, and file-backed
stream storage inherited by one consumer per work kind. Replicas bind to the
same durable. The Worker checks the registered filter and delivery settings at
startup and fails closed if the stream/durable is absent or drifted; it does not
create default consumers.

## Processing and recovery

- The database lease acquisition updates `QUEUED` to `RUNNING`, increments
  attempts, and assigns a 120-second lease plus a random UUID lease token. Long
  handlers renew the lease and send JetStream progress ACKs every 30 seconds.
- Completion or terminal failure must match the lease token. A stale worker
  cannot overwrite another worker's result. Business side effects are
  idempotent: QSO keys and snapshots are unique/upserted, notifications use
  dedupe keys, and a PDF writes to the issuance's stable object key.
- On transient failure, Activity records a safe exception class, returns the job
  to `QUEUED` with capped exponential `available_at`, and NAKs with delay. When
  a delivery arrives before the database retry/lease is ready, the worker keeps
  the message pending and extends the ACK timer instead of spending a broker
  delivery attempt on not-yet-eligible work.
- At the eighth failed handler delivery, the worker transactionally marks the
  job `FAILED` and inserts an `activity_work_dead_letter` record before ACK.
  The DLQ stores bounded identifiers/category, not full source payloads.
- An operator uses `redrive_activity_job.py --job-id <uuid> --actor <name>
  --reason <reviewed-reason>` from an authorized Activity service context. The
  database transaction resets the failed job, creates a fresh outbox event ID
  for the same stable work ID, writes `activity_work_redrive_audit`, and marks
  prior matching work-DLQ records resolved. The standard outbox relay republishes
  it; there is no direct broker publish path.
- Invalid envelopes and missing/mismatched job rows are recorded in the
  Activity work DLQ and terminated. If a broker-level delivery limit is reached
  while the owning database is unavailable, the job remains the recovery
  authority; inspect the durable and queued job age, then use audited database
  redrive after restoring the database connection.
- `NOTIFICATION_SEND` is not a queue command. The notification projection is
  inserted as `DELIVERED` in its owner transaction. Do not create a provider
  command unless external notification delivery becomes a real side effect.

## Schema retirement and scheduled work

Migration `007_activity_jetstream_work.sql` backfills only queued selected jobs,
refuses a first cutover while a selected job is running, and drops the old
`activity_job_claim_idx` used by the DB poller. `activity_job` remains because
it serves API job status, idempotency, recovery and metrics. The replacement
`activity_job_status_kind_idx` supports the durable job views and compatibility
reconciliation. Migration also deletes only synthetic `NOTIFICATION_SEND`
job rows after the corresponding in-app notifications are marked `DELIVERED`.

ADIF object retention stays on the Activity CronJob and remains restricted to
terminal old imports. It is scheduled cleanup, not accepted-work dispatch.
Any separately scheduled reconciliation remains an owning database recovery
mechanism; it does not claim or execute JetStream work. The rollout-only legacy
outbox repair was disabled after all old API replicas were gone and it had
repaired zero rows.

## Metrics and rollout checks

The Activity API exposes bounded database-derived metrics by job kind/status:
queued age, completed latency, retry total, failed jobs, and unresolved work-DLQ
count. Operations' Grafana dashboard adds job status/kind, age, latency/retries,
and unresolved-DLQ panels. Do not add job IDs or other unbounded labels.

Phase 4 production verification found the WorkQueue stream and six exact
durables ready, no selected queued/running job rows, no pending or ack-pending
messages, and active worker pull waiters. The compatibility repair is disabled
and `activity_job_claim_idx` is absent. The Activity API exports per-kind job
status, queue age, retries, latency observations when jobs complete, and
unresolved DLQ counts; Prometheus and Operations dashboards use these bounded
metrics. No latency sample exists until real jobs execute. Preserve
`MYOTA_EVENTS` Interest retention and all Geodata work durables until their
separate planned phases.
