# Phase 5 Geodata work migration evidence — 10 October 2026

## Status

Phase 5 is in production cutover and qualification. Helm revision 187 applied the
Geodata WorkQueue route and the new immutable Geodata/runtime image digests. All
four Geodata workers now subscribe to their target subjects in
`MYOTA_GEODATA_WORK`; migration 021 is installed and the recovery columns and
indexes are present. The previous production route remains provisioned but
inactive and empty for rollback.

The follow-up partial-cascade recovery fix is now in the deployed Geodata image.
A failure after the Activity cascade no longer becomes a terminal Geodata failure
that is ACKed: the handler persists an expired processing lease and rethrows so
JetStream retries; the database repair scanner can reconstruct the command
after its age bound. The isolated test covers Activity success followed by a
Geodata-side failure and a successful retry. A full cross-service test using two
disposable databases and both real service handlers remains open.

The latest rollout began at 20:55:08 UTC on 10 October. Keep the four legacy
Geodata durables through the 24-hour rollback observation, ending no earlier
than 20:55:08 UTC on 11 October. Remove only those four obsolete durable
consumers after confirming the target route, database recovery and no legacy
backlog. Do not remove the shared `MYOTA_EVENTS` stream, its Activity
notification consumer, Geodata source tables, work rows, outbox or recovery
columns/indexes.

## Ownership and boundaries

- `myota-geodata-service` owns Geodata domain rows, the outbox transaction,
  worker handlers, leases, cancellation and recovery migration.
- `myota-deploy` owns Helm/Fleet provisioning, shared outbox relay, deployment
  values and deployment migration copies.
- `myota-platform` synchronizes the deployment/runtime copies. The Geodata
  service repository remains authoritative for its dedicated worker image.
- `myota-contracts` owns registered work contracts and payload projections.
- PostgreSQL remains the source of truth. Operations retains read-only broker
  inspection. JetStream is a bounded work transport, not an event archive.
- Off-node broker recovery remains explicitly deferred for this single-node
  deployment. Production data is not used for failure injection.

## Deployed topology and live observations

| Component | Observed state |
|---|---|
| Helm/Fleet | Helm revision 187; migration and JetStream provisioner jobs completed. All application pods became ready on the pinned image digests. Fleet was still reporting `WaitApplied` while the reconciliation completed. |
| Geodata API/worker image | `ghcr.io/myota-platform/myota-geodata-service@sha256:c6f5dee746579469ba827a4af78741acbe5d25e825471c84c95ec5574e6e2aee`. The source commit's immutable tag and `latest` resolved to this same digest. |
| Shared runtime image | `ghcr.io/myota-platform/myota-service@sha256:1f4002619cee64d9d05f06b96c93d348df5ae725806d08da383d74e0c84c91e8`. The deploy commit tag and `latest` resolved to this same digest. |
| `MYOTA_EVENTS` | File storage, Interest retention, zero messages. Activity's notification durable remains live. Four old Geodata durable definitions remain for the rollback window with zero pending, ack-pending and redelivery counts. |
| `MYOTA_GEODATA_WORK` | File storage, one replica, WorkQueue retention, configured finite 30-day/3-GiB/500k-message/1-MiB limits, zero messages. |
| Replacement consumers | Exact pull filters: `geodata-preprocessing-v1` → `myota.work.geodata.import-preprocess.v1`; `geodata-import-promotion-v1` → `myota.work.geodata.import-promotion.v1`; `geodata-entity-deletion-v1` → `myota.work.geodata.entity-delete.v1`; `geodata-location-enrichment-v1` → `myota.work.geodata.location-enrichment.v1`. All use explicit ACK, bounded pending and configured retry limits (100 deliveries, except location enrichment at 8). All four had zero pending, ack-pending and redelivered messages at inspection. |
| Worker subscriptions | Live logs show all four replacement subjects. The workers no longer subscribe to the legacy Geodata work subjects. |
| Database migration | Migration job 187 completed. Read-only inspection found `work_dispatched_at` on `import_run`, `geodata_import_processing_queue`, and `geodata_entity`, plus `import_run_work_recovery_idx`, `import_queue_work_recovery_idx`, and `geodata_location_work_recovery_idx`. Active queued/processing rows without dispatch timestamps: zero. |
| Production domain work | No accepted Geodata work was available for a production processing test. No production row, outbox record, message, or dead letter was created, redriven, resolved, or deleted during this qualification. Six historical DLQs map to cancelled imports and remain untouched. |

## Changes and source commits

- Geodata source recovery fix: `myota-geodata-service` commit
  [`0a3c9e1`](https://github.com/myota-platform/myota-geodata-service/commit/0a3c9e199b7ea7754ef1be3244f60fecb25b0c75).
- Geodata retry regression test: `myota-geodata-service` commit
  [`29b3cda`](https://github.com/myota-platform/myota-geodata-service/commit/29b3cda6b00510f32b176a2a40d493790e9377fb).
- Deploy runtime mirror: `myota-deploy` commit
  [`5a758f4`](https://github.com/myota-platform/myota-deploy/commit/5a758f4dd88f49854afe2bd3ed7d2212a4958a6b).
- Platform runtime mirror: `myota-platform` commit
  [`37be00c`](https://github.com/myota-platform/myota-platform/commit/37be00c1273d937806c795f231f01e87dbd7546e).
- Deploy digest pins: `myota-deploy` commit
  [`c1a4e60`](https://github.com/myota-platform/myota-deploy/commit/c1a4e609d2cb9166b873a8eaa263d66f378b2035).
- Platform digest mirror: `myota-platform` commit
  [`c8137fd`](https://github.com/myota-platform/myota-platform/commit/c8137fd5310ed904611aa2394a074e75234bbd3a).

The corresponding source tag and registry digest were verified before updating
the Deploy pin. The values file is byte-identical in Deploy and Platform.
Runtime migrations are additive; no unused Phase 5 database object was
identified for retirement. The recovery columns and indexes are required by
the live repair worker and must remain.

## Focused verification

| Check | Result | Evidence and limit |
|---|---|---|
| Geodata suite | Pass | 154 tests in 59.849 seconds; one optional two-process test skipped because that separate setup was not configured. Includes database-backed PostGIS/JetStream tests against the disposable namespace. |
| Geodata lint/format | Pass | Ruff check and format check on `geodata.py`, `tests/test_geometry.py`, and `tests/test_jetstream_worker_delivery.py`. |
| Deploy relay/topology | Pass | 25 tests in `test_outbox_contracts` and `test_jetstream_topology`; Ruff and formatting passed on the mirrored deletion handler. |
| Database connection failure | Pass | In the disposable host-K3s test database, the first handler connection was refused on local test port 1; JetStream NAKed and redelivered the message, the next database connection succeeded, and the durable settled at zero pending and ack-pending. No false ACK occurred. |
| Partial Activity/Geodata deletion | Pass, focused unit test | Activity cascade success was followed by an injected Geodata-side failure; the work remained retryable, and a subsequent delivery completed. It does not yet prove two real service/database instances end to end. |
| Duplicate/lost ACK and commit boundary | Pass, isolated | Existing JetStream integration confirms redelivery after commit is idempotent and ACK follows durable state. |
| Long handler/ACK wait | Pass, isolated | Heartbeat test held a handler beyond its configured ACK wait without redelivery. |
| Stale-row repair | Pass, database-backed | Each of preprocessing, promotion, deletion and location work was recreated once through the outbox; an immediate second scan emitted no duplicate. Recovery batch size is 50 and minimum work age is 300 seconds. |
| Expiry/replay | Pass in separate checks | Private stream age expiry and database stale-row redispatch each passed. The expiry-to-republish-to-completion chain remains open. |
| Durable recreation | Pass, isolated | An unacked command survived consumer recreation and was ACKed after redelivery. |
| Broker restart/restore | Pass, isolated, same node only | A file-backed WorkQueue message and durable state survived a NATS Pod restart using the same PVC, then were ACKed. Off-node restore is out of scope. The test-only namespace contains no production data. |
| Production route/schema | Pass, read-only | Four live worker logs use target subjects; exact target durables and migration 021 markers were inspected; both streams had zero messages. Production processing was not claimed because there was no accepted work. |
| GitHub Actions status | Not recorded | The GitHub connector returned no combined status contexts for the direct-main commits. Local tests and live image/digest checks are recorded; confirm repository Actions checks before final phase closure. |

## Remaining gates

1. Complete the full 24-hour rollback observation after the latest rollout. Then
   recheck zero legacy pending/ack-pending/redelivered messages, exact replacement
   filters, migration markers, outbox/row recovery age, and Fleet readiness.
2. Remove only the four legacy Geodata durable consumers from `MYOTA_EVENTS`
   after the observation and the safe rollback check. Preserve Activity's
   notification durable and the shared stream.
3. Run a disposable two-database cross-service test of Activity cascade success,
   Geodata failure, JetStream retry/database repair, and final completion. Include
   cancellation racing with a real acknowledged delivery and connect message
   expiry to successful owner-row reconstruction and completion.
4. Record successful GitHub Actions checks for the two service commits, deploy
   image and mirrored values commit, then confirm their source links remain valid.
5. Refresh Phase 5 plan checkboxes and report cleanup after all gates pass.

## Schema retirement and cleanup

No Phase 5 database schema object is obsolete. Keep migration 021's dispatch
columns/indexes and the outbox, source jobs, cancellation/history records,
checkpoints, and dead letters; these are the durable recovery boundary. Do not
purge the six historical cancelled-import dead letters.

The isolated namespace `myota-phase5-validation` currently contains only NATS
and PostGIS test services. Remove the namespace, test PVC, and local port
forwards after the remaining isolated cross-service checks. No production test
data was created.
