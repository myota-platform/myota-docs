# Phase 5 Geodata work migration evidence — 10 October 2026

## Status

Phase 5 is in production cutover and qualification. Helm revision 189 is
deployed. Fleet reports `Ready=True` at commit
`bbb3276296c8fa86941315b582caa76e178bd94f`, with 60/60 resources ready.
The file-backed Geodata WorkQueue route, four target subscriptions, and
migration 021 are live; the former production route remains provisioned but
inactive and empty for rollback.

The deployed Geodata image contains the partial-cascade recovery fix. A failure
after the Activity cascade leaves an expired processing lease and rethrows so
JetStream retries; the bounded database repair scanner can reconstruct the
command after its age bound. A disposable two-database test now uses the real
Activity API/database, Geodata handler/database, and private JetStream stream.
An injected Geodata-side failure after Activity commits is NAKed; redelivery
completes deletion, with exactly one Activity cascade outbox fact and no
remaining stream message or pending delivery. This test exposed that Activity
was not persisting the request `Idempotency-Key`; that source bug is fixed and
committed, but its new image has not yet been published/deployed.

The Geodata rollback observation began at 20:58:36 UTC on 10 October, when Helm
revision 189 became deployed. Keep the four legacy Geodata durables through
24 hours, ending no earlier than 20:58:36 UTC on 11 October. Remove only those
four durable consumers after confirming the target route, database recovery,
Fleet readiness, and no legacy backlog. Do not remove the shared
`MYOTA_EVENTS` stream, its Activity notification consumer, Geodata source
tables, work rows, outbox or recovery columns/indexes.

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
| Helm/Fleet | Helm revision 189 is deployed. Fleet `Ready=True` at deploy commit `bbb3276296c8fa86941315b582caa76e178bd94f`; 60/60 resources ready at inspection. |
| Geodata API/worker image | `ghcr.io/myota-platform/myota-geodata-service@sha256:c6f5dee746579469ba827a4af78741acbe5d25e825471c84c95ec5574e6e2aee`; live pods match this digest. |
| Shared runtime image | `ghcr.io/myota-platform/myota-service@sha256:1f4002619cee64d9d05f06b96c93d348df5ae725806d08da383d74e0c84c91e8`; live outbox pods match this digest. |
| `MYOTA_EVENTS` | File storage, Interest retention, zero messages. Activity's notification durable remains live. Four old Geodata durable definitions remain for the rollback window with zero pending, ack-pending and redelivery counts. |
| `MYOTA_GEODATA_WORK` | File storage, one replica, WorkQueue retention, configured finite 30-day/3-GiB/500k-message/1-MiB limits, zero messages. |
| Replacement consumers | Exact pull filters: `geodata-preprocessing-v1` → `myota.work.geodata.import-preprocess.v1`; `geodata-import-promotion-v1` → `myota.work.geodata.import-promotion.v1`; `geodata-entity-deletion-v1` → `myota.work.geodata.entity-delete.v1`; `geodata-location-enrichment-v1` → `myota.work.geodata.location-enrichment.v1`. All use explicit ACK, bounded pending and configured retry limits (100 deliveries, except location enrichment at 8). All four had zero pending, ack-pending and redelivered messages at inspection. |
| Worker subscriptions | Live logs show all four replacement subjects. The workers no longer subscribe to the legacy Geodata work subjects. |
| Database migration | Migration job 187 completed. Read-only inspection found `work_dispatched_at` on `import_run`, `geodata_import_processing_queue`, and `geodata_entity`, plus `import_run_work_recovery_idx`, `import_queue_work_recovery_idx`, and `geodata_location_work_recovery_idx`. Active queued/processing rows without dispatch timestamps: zero. |
| Activity API image | Live Activity API/worker pods still use `ghcr.io/myota-platform/myota-activity-service@sha256:f46c10c286ed82c15bed37dc84f9152a782403198ae0332c8cf04bf75067b023`. The new Activity idempotency fix is source-committed but is not yet present in this deployed image. |\n| Production domain work | No accepted Geodata work was available for a production processing test. No production row, outbox record, message, or dead letter was created, redriven, resolved, or deleted. Six historical DLQs map to cancelled imports and remain untouched. |

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

The source tag/digest and values mirrors were verified before the Deploy pin.
No unused Phase 5 database object was identified for retirement. Migration
021's dispatch columns and indexes remain required by the live repair worker.
Activity cascade retry idempotency is committed in the authoritative
`myota-activity-service` repository and synchronized to `myota-deploy` and
`myota-platform`: handler `cdea2ba`, repository transaction
`d364932`, regression test `6448fc2`; mirror commits are
`b76f825`/`259e2a7`/`84764e9` in Deploy and
`6dbab48`/`9953b00` in Platform. GitHub combined status queries returned no
status contexts for these Activity commits, so CI and image publication still
need confirmation.

## Focused verification

| Check | Result | Evidence and limit |
|---|---|---|
| Geodata suite | Pass | 154 tests in 59.849 seconds; one optional two-process test skipped because that separate setup was not configured. Includes database-backed PostGIS/JetStream tests against the disposable namespace. |
| Geodata lint/format | Pass | Ruff check and format check on `geodata.py`, `tests/test_geometry.py`, and `tests/test_jetstream_worker_delivery.py`. |
| Deploy relay/topology | Pass | 25 tests in `test_outbox_contracts` and `test_jetstream_topology`; Ruff and formatting passed on the mirrored deletion handler. |
| Database connection failure | Pass | In the disposable host-K3s test database, the first handler connection was refused on local test port 1; JetStream NAKed and redelivered the message, the next database connection succeeded, and the durable settled at zero pending and ack-pending. No false ACK occurred. |
| Activity cascade idempotency | Pass, isolated | The durable repository serializes same-key requests with a transaction advisory lock and stores the response atomically with the cascade fact. Five rounds of eight concurrent requests each produced one response, one idempotency row and one outbox fact. Activity's 40-test suite passed (one optional JetStream broker test skipped); Ruff and formatting checks passed. The source fix is not yet deployed in the Activity image. |
| Two-database cascade/JetStream retry | Pass, isolated host-K3s | Real Activity API and Activity PostgreSQL plus Geodata handler and PostGIS ran with a private file-backed WorkQueue and durable pull consumer. Injected Geodata failure after Activity commit produced NAK/redelivery; completion removed the entity, wrote exactly one `activity.entity.cascade-deleted.v1` outbox row, and left zero stream messages, pending messages or ack-pending messages. Disposable fixture rows, idempotency entry, event row and private stream were removed. |
| Cancellation during acknowledged work | Pass, isolated host-K3s | The private preprocessing durable claimed a disposable import, the cancellation API persisted CANCELLING while work was in flight, and the worker finalized CANCELLED before ACK. One cancellation fact was durable; the stream ended with zero messages/pending/ack-pending. No production row or feature data was written. Confirmed deletion jobs are not cancellable. |
| Expiry/replay/recompletion | Pass, connected disposable chain | A private WorkQueue message expired before consumption; the age-bounded Geodata scanner rebuilt a command from the processing owner row through the transactional outbox; a durable redelivery completed the Activity cascade and Geodata deletion. The stream ended with zero messages, pending and ack-pending. |
| Duplicate/lost ACK and commit boundary | Pass, isolated | Existing JetStream integration confirms redelivery after commit is idempotent and ACK follows durable state. |
| Long handler/ACK wait | Pass, isolated | Heartbeat test held a handler beyond its configured ACK wait without redelivery. |
| Stale-row repair | Pass, database-backed | Each of preprocessing, promotion, deletion and location work was recreated once through the outbox; an immediate second scan emitted no duplicate. Recovery batch size is 50 and minimum work age is 300 seconds. |
| Expiry/replay | Pass in separate checks | Private stream age expiry and database stale-row redispatch each passed. The expiry-to-republish-to-completion chain remains open. |
| Durable recreation | Pass, isolated | An unacked command survived consumer recreation and was ACKed after redelivery. |
| Broker restart/restore | Pass, isolated, same node only | A file-backed WorkQueue message and durable state survived a NATS Pod restart using the same PVC, then were ACKed. Off-node restore is out of scope. The test-only namespace contains no production data. |
| Production route/schema | Pass, read-only | Four live worker logs use target subjects; exact target durables and migration 021 markers were inspected; both streams had zero messages. Production processing was not claimed because there was no accepted work. |
| GitHub Actions and Activity image | Open | GitHub combined-status queries returned no contexts for Activity source commits. The Activity container still runs its previous digest; verify CI, published image, immutable digest pin, and production rollout before closing. |

## Remaining gates

1. Complete the 24-hour Geodata rollback observation after Helm revision 189.
   Recheck zero legacy pending/ack-pending/redelivered messages, exact replacement
   filters, migration markers, owner-row recovery age and Fleet readiness. Do
   not retire a durable before 20:58:36 UTC on 11 October 2026.
2. Remove only the four legacy Geodata durable consumers from `MYOTA_EVENTS`
   after the observation and safe rollback check. Preserve Activity's
   notification durable and the shared stream.
3. Confirm GitHub Actions for the Activity source commits, publish the
   corrected Activity image, pin its immutable digest, and verify Fleet's
   rollout. Keep Phase 5 open until this source fix is live.
4. Shut down the local Activity API and port-forwards, then remove the isolated
   K3s namespace/PVC after final read-only cleanup checks.

## Schema retirement and cleanup

No Phase 5 database schema object is obsolete. Keep migration 021's dispatch
columns/indexes and the outbox, source jobs, cancellation/history records,
checkpoints, and dead letters; these are the durable recovery boundary. Do not
purge the six historical cancelled-import dead letters.

The isolated namespace `myota-phase5-validation` contains disposable NATS,
Geodata PostGIS, and Activity PostgreSQL services plus a local Activity API
process and port-forwards. Test fixtures, idempotency/outbox rows, checkpoints,
and all private JetStream streams were removed after the checks. Stop the local
process and port-forwards, then delete the namespace/PVC. No production data
was used or modified.
