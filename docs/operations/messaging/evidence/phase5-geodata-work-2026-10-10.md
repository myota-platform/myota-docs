# Phase 5 Geodata work migration evidence — 10 October 2026

## Current status — 11 October 2026

Helm revision 193 completed successfully at 22:00:02 UTC on 10 October. Fleet
reports `Ready=True` at Deploy commit
`cfecd655d9c0eee9d19db26725fb11c99366815a` as of 22:03:29 UTC, and Helm reports
revision 193 deployed. Migration Job 193 completed; all MyOTA Deployments are
ready. Live first-party image references are digest-pinned: Activity
`f764bfe7193ba8166c84f3d6b063547f94fcc17b6c819aa597c617ed6c835261), Geodata
`215942e8f728fd7fcb5a1f9630045eba96ca51a66f2ac20cc5db7dfd6c879243), Identity
`bd8b6b507614c4924a12061051b456690ca68bc66740dbc0f7a4e907ca1175c7),
Programme
`891c02d3a94ce751a52fdeda90e356d2319639e419642d0ce44686489bd7a765), and
shared runtime
`56709d36f8e9ab6d39a4da56b4dc9f28269e34a3973238deb955a0ac78e82555).

The latest read-only production broker sample, about 22:05 UTC, found
`MYOTA_EVENTS` file-backed with Interest retention, zero messages/bytes, and
Activity plus all four legacy Geodata durables at zero pending, ack-pending and
redelivered. `MYOTA_GEODATA_WORK` remains file-backed WorkQueue with zero
messages/bytes; all four target filters are exact, counters are zero, and each
target durable has one waiting pull. No production message or data was written.

A new disposable validation namespace exercised synthetic stale-owner recovery
and JetStream delivery against isolated PostGIS databases and an isolated
in-memory broker. The recovery test created one synthetic stale row for each of
preprocessing, promotion, deletion and location enrichment; one scan recreated
four outbox commands, and a second scan recreated none. The 10-test delivery
suite was run twice. Each run had one intermittent failure where the test read
`num_ack_pending=1` immediately after the asynchronous ACK; the ACK-only test
passed on a focused rerun. A post-suite broker inspection found all 21 private
test streams at zero messages, pending, ack-pending and redelivered. Record the
suite as behavior evidence with a flaky immediate-counter assertion, not as a
clean suite pass. Test rows/outbox fixtures were removed by the suite and
namespace deletion; `myota-phase5-validation` is verified absent.

The observation anchor is revision 193's fully ready rollout at 22:03:29 UTC on
10 October and ends no earlier than 22:03:29 UTC on 11 October. The earlier
revision 192 snapshot below is historical and no longer controls the gate.

## Snapshot at revision 192 — 10 October 2026

| Component | Observed state |
|---|---|
| Helm/Fleet | Helm revision 192 is deployed. Fleet GitRepo `myota-deploy` is `Ready=True` at deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a`; the MyOTA BundleDeployment reports 60/60 resources Ready. |
| Activity API/worker/notification | Pod template image reference and live pod ImageID are `ghcr.io/myota-platform/myota-activity-service@sha256:f764bfe7193ba8166c84f3d6b063547f94fcc17b6c819aa597c617ed6c835261`. This image includes transactional `Idempotency-Key` handling. |
| Geodata API/worker | Pod template image reference and live pod ImageID are `ghcr.io/myota-platform/myota-geodata-service@sha256:c6f5dee746579469ba827a4af78741acbe5d25e825471c84c95ec5574e6e2aee`; includes partial-cascade recovery. |
| Shared runtime/outbox | Pod template image reference and live pod ImageID are `ghcr.io/myota-platform/myota-service@sha256:1f4002619cee64d9d05f06b96c93d348df5ae725806d08da383d74e0c84c91e8`. |
| `MYOTA_EVENTS` | File storage, Interest retention, subjects `myota.events.>` and `myota.geodata.>`, 30-day max age, no configured byte/message cap, zero messages/bytes. Activity notification durable remains active. Four legacy Geodata durables remain present, all zero pending/ack-pending/redelivered and no waiting worker. |
| `MYOTA_GEODATA_WORK` | File-backed, one replica, WorkQueue, subject `myota.work.geodata.>`, 30-day max age, 3 GiB, 500,000 messages, 1 MiB per message; zero messages/bytes. |
| Replacement consumers | Four exact pull filters: preprocessing `myota.work.geodata.import-preprocess.v1`, promotion `myota.work.geodata.import-promotion.v1`, deletion `myota.work.geodata.entity-delete.v1`, location `myota.work.geodata.location-enrichment.v1`. Explicit ACK; max delivery is 100 except location at 8. All have zero pending, ack-pending and redelivered; each has one waiting pull worker. |
| Legacy consumers | `geodata-entity-deletion-v1` → `myota.geodata.entity.delete.v1`; `geodata-import-processing-v2` → `myota.geodata.import.process.v1`; `geodata-location-enrichment-v1` → `myota.geodata.entity.location-enrichment.v1`; `geodata-preprocessing-v1` → `myota.geodata.import.preprocess.v1`. All four remain inactive with zero pending/ack-pending/redelivered. |
| Database migration | Helm migration Job 191 completed. Migration 021 markers remain present: dispatch timestamps on `import_run`, `geodata_import_processing_queue`, and `geodata_entity`; indexes `import_run_work_recovery_idx`, `import_queue_work_recovery_idx`, and `geodata_location_work_recovery_idx`. Previously inspected live un-dispatched queued/processing count was zero. |
| Production work | No accepted Geodata work was available for a production processing test. No production row, outbox record, message or dead letter was created, redriven, resolved or deleted. Six historical DLQs map to cancelled imports and remain untouched. |

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

Migration 021's dispatch columns/indexes remain required by the live repair
worker. No Phase 5 schema object is obsolete. Activity cascade retry
idempotency is committed in the authoritative `myota-activity-service`
repository and synchronized to Deploy/Platform: handler `cdea2ba`,
repository `d364932`, regression test `6448fc2`; mirror commits are
`b76f825`/`259e2a7`/`84764e9` in Deploy and
`6dbab48`/`9953b00` in Platform. The Activity image was published and
deployed at immutable digest `f764bfe…`; the local 40-test suite passed
(one optional broker test skipped). GitHub combined-status queries returned no
status contexts, so no separate Actions run summary is claimed.

The immutable image-reference fix is committed directly to main in Deploy at
[`a68eedd`](https://github.com/myota-platform/myota-deploy/commit/a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109)
and mirrored to Platform at
[`a184baa`](https://github.com/myota-platform/myota-platform/commit/a184baac3f36e0272cbc79107f4b362139de7515).


## Focused verification

| Check | Result | Evidence and limit |
|---|---|---|
| Geodata suite | Pass | 154 tests in 59.849 seconds; one optional two-process test skipped because that separate setup was not configured. Includes database-backed PostGIS/JetStream tests against the disposable namespace. |
| Geodata lint/format | Pass | Ruff check and format check on `geodata.py`, `tests/test_geometry.py`, and `tests/test_jetstream_worker_delivery.py`. |
| Deploy relay/topology | Pass | 25 tests in `test_outbox_contracts` and `test_jetstream_topology`; Ruff and formatting passed on the mirrored deletion handler. |
| Database connection failure | Pass | In the disposable host-K3s test database, the first handler connection was refused on local test port 1; JetStream NAKed and redelivered the message, the next database connection succeeded, and the durable settled at zero pending and ack-pending. No false ACK occurred. |
| Activity cascade idempotency | Pass, isolated and live | Five rounds of eight concurrent same-key requests produced one response, idempotency row and cascade outbox fact. Activity's 40-test suite passed (one optional broker test skipped); Ruff and formatting passed. The published digest is deployed to the API, workers and notification consumer. |
| Two-database cascade/JetStream retry | Pass, isolated host-K3s | Real Activity API and Activity PostgreSQL plus Geodata handler and PostGIS ran with a private file-backed WorkQueue and durable pull consumer. Injected Geodata failure after Activity commit produced NAK/redelivery; completion removed the entity, wrote exactly one `activity.entity.cascade-deleted.v1` outbox row, and left zero stream messages, pending messages or ack-pending messages. Disposable fixture rows, idempotency entry, event row and private stream were removed. |
| Cancellation during acknowledged work | Pass, isolated host-K3s | The private preprocessing durable claimed a disposable import, the cancellation API persisted CANCELLING while work was in flight, and the worker finalized CANCELLED before ACK. One cancellation fact was durable; the stream ended with zero messages/pending/ack-pending. No production row or feature data was written. Confirmed deletion jobs are not cancellable. |
| Expiry/replay/recompletion | Pass, connected disposable chain | A private WorkQueue message expired before consumption; the age-bounded Geodata scanner rebuilt a command from the processing owner row through the transactional outbox; a durable redelivery completed the Activity cascade and Geodata deletion. The stream ended with zero messages, pending and ack-pending. |
| Duplicate/lost ACK and commit boundary | Pass, isolated | Existing JetStream integration confirms redelivery after commit is idempotent and ACK follows durable state. |
| Long handler/ACK wait | Pass, isolated | Heartbeat test held a handler beyond its configured ACK wait without redelivery. |
| Stale-row repair | Pass, database-backed | Each of preprocessing, promotion, deletion and location work was recreated once through the outbox; an immediate second scan emitted no duplicate. Recovery batch size is 50 and minimum work age is 300 seconds. |
| Expiry/replay/recompletion | Pass, connected isolated chain | A message expired, the owner-row recovery scanner republished through the transactional outbox, and redelivery completed the cascade with an empty private stream. |
| Durable recreation | Pass, isolated | An unacked command survived consumer recreation and was ACKed after redelivery. |
| Broker restart/restore | Pass, isolated, same node only | A file-backed WorkQueue message and durable state survived a NATS Pod restart using the same PVC, then were ACKed. Off-node restore is out of scope. The test-only namespace contains no production data. |
| Production route/schema | Pass, read-only | Four live worker logs use target subjects; exact target durables and migration 021 markers were inspected; both streams had zero messages. Production processing was not claimed because there was no accepted work. |
| Image publication, pin and rollout | Pass with evidence limit | Activity digest `f764bfe…` is published, configured by digest, and matches live pod references/ImageIDs. The Deploy workflow's explicit run summary was not returned by the connector; the local Activity 40-test suite passed (one optional broker test skipped). |

## Remaining gates

1. Keep the four legacy Geodata durables through the 24-hour rollback
   observation, which ends no earlier than 21:39:22 UTC on 11 October 2026.
   Recheck zero pending/ack-pending/redelivered, exact target filters, schema
   markers, owner-row recovery age and Fleet readiness.
2. After those checks pass, retire only the four named legacy Geodata durables
   from `MYOTA_EVENTS`. Preserve the Activity notification durable, shared
   stream, Geodata source tables, work rows, outbox/recovery history and all
   Phase 5 database recovery columns/indexes.

## In-window production observation — 10 October 2026, 21:52 UTC

This is an early read-only sample, about 13 minutes after the revision 192
rollout. It does not complete or shorten the required 24-hour observation.

- Helm history still reports revision 192 deployed, completed at 21:39:22 UTC.
  Fleet GitRepo `myota-deploy` reports Ready=True at
  `6443473828305ab9d02a918bbe990d01abe97f6a`; the MyOTA bundle is deployed
  and monitored.
- A metadata-only JetStream query through the existing `myota-geo-outbox`
  pod made no publish, consume, ACK, purge, or configuration request.
  `MYOTA_EVENTS` remains file-backed Interest retention with subjects
  `myota.events.>` and `myota.geodata.>`; it reported zero messages and
  bytes. Activity's `activity-notifications-v1` durable had zero pending,
  ack-pending, and redelivered messages, with one waiting pull.
- `MYOTA_GEODATA_WORK` remains file-backed WorkQueue and reported zero
  messages and bytes. All four target durables had their exact registered
  filters, zero pending/ack-pending/redelivered, and one waiting pull each.
- The four legacy durables remain present with their expected old filters and
  zero pending/ack-pending/redelivered, with no waiting workers. The disposable
  validation namespace remains absent.

The observation remains open until at least 21:39:22 UTC on 11 October 2026.
A later Helm rollout/restart resets the observation anchor to that rollout's
completion time.

## Synthetic validation and revised production sample — 11 October 2026

- The test ran only in `myota-phase5-validation`, with temporary PostGIS
  databases and a dedicated in-memory JetStream broker. The platform migration
  runner applied the current schema with data-copy disabled.
- Recovery recreated one command for each of the four Geodata work kinds through
  the transactional outbox, and an immediate second scan emitted none.
- The delivery suite exercised ACK completion, competing pull consumers,
  database failure and recovery, durable recreation, heartbeat beyond ACK wait,
  lost ACK idempotency, graceful shutdown, age expiry, envelope identity and
  stale-row recovery. Two full-suite runs each had one timing-sensitive final
  counter assertion (ACK had returned but broker `num_ack_pending` had not yet
  settled); the focused ACK test passed and the later broker inspection showed
  zero counters on every private stream. This does not count as a clean full
  suite pass and does not invalidate the independent earlier Phase 5 checks.
- After test completion, all 21 private streams reported zero messages,
  pending, ack-pending and redelivered. The namespace was deleted and verified
  absent. No production writes were made.
- Production revision 193 is deployed and Fleet Ready. At about 22:05 UTC,
  read-only stream inspection found zero messages/bytes in both streams; the
  four target Geodata durables had their registered filters, zero counters and
  one waiter each. The four legacy durables remained present with zero
  pending/ack-pending/redelivery and no waiting workers. Activity's notification
  durable remained present and healthy.

The observation remains open until at least 22:03:29 UTC on 11 October 2026.
A later production rollout or restart resets the anchor. Do not retire a legacy
durable before the final checks.

## Post-observation retirement procedure — prepared, not executed

After the 24-hour gate has elapsed, perform the final checks in this order:

1. Confirm no later Helm rollout has reset the observation clock. Require Helm
   revision 193 to remain deployed, Fleet Ready=True at the expected commit,
   the MyOTA bundle monitored/deployed, and all MyOTA Deployments ready.
2. Inspect both streams and all eight work durables read-only. Confirm the four
   legacy durable names and filters still match the table above and each has
   zero pending, ack-pending, and redelivered messages. Confirm the four target
   filters and active pull workers remain correct. Check the migration 021
   markers and documented owner-row recovery/backlog queries. If any check is
   unexpected or fails, stop and preserve every legacy durable.
3. Open a temporary local port-forward:
   `kubectl -n myota port-forward service/myota-nats 14222:4222`.
   In another terminal, use the official [NATS CLI](https://github.com/nats-io/natscli)
   against `nats://127.0.0.1:14222`. Inspect and remove one consumer at a
   time, accepting the CLI's confirmation prompt only for these four exact
   names. Do not use a force flag:
   `nats --server nats://127.0.0.1:14222 consumer rm MYOTA_EVENTS geodata-entity-deletion-v1`
   `nats --server nats://127.0.0.1:14222 consumer rm MYOTA_EVENTS geodata-import-processing-v2`
   `nats --server nats://127.0.0.1:14222 consumer rm MYOTA_EVENTS geodata-location-enrichment-v1`
   `nats --server nats://127.0.0.1:14222 consumer rm MYOTA_EVENTS geodata-preprocessing-v1`
4. After each removal, verify that exact legacy consumer is absent before
   proceeding. If removal or verification fails, stop. Finally verify
   `MYOTA_EVENTS` and `activity-notifications-v1` still exist and the target
   Geodata WorkQueue durables remain intact. Stop the port-forward and record
   UTC timestamps, command results, final counters, Fleet status, migration
   markers, and whether recovery rows were due.
5. Never delete or purge `MYOTA_EVENTS`, its Activity durable, any target
   work durable, PostgreSQL tables/columns/indexes, work/outbox/source rows,
   checkpoints, or dead letters as part of this cleanup.

## Schema retirement and cleanup

No Phase 5 database schema object is obsolete. Keep migration 021's dispatch
columns/indexes and the outbox, source jobs, cancellation/history records,
checkpoints, and dead letters; these are the durable recovery boundary. Do not
purge the six historical cancelled-import dead letters.

Cleanup is complete. The local Activity API process and port-forwards were
stopped; private streams, disposable test fixtures and namespace/PVC
`myota-phase5-validation` were deleted and verified absent. No production
data or production JetStream messages were used for failure injection.
