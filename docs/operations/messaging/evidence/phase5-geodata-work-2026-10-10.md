# Phase 5 Geodata work migration evidence — 10 October 2026

## Status

Phase 5 production cutover is healthy and its image deployment is digest
immutable. Helm revision 192 completed at 21:39:22 UTC on 10 October. Fleet
reports `Ready=True` at deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a`; its MyOTA BundleDeployment
reports 60/60 resources Ready. Activity, Geodata and shared runtime pods
reference their configured digests directly, and live pod ImageIDs match. The
Activity idempotency fix is live. Revision 192 restarted pods with the same
configured image digests as revision 191 and is the current rollback-observation
anchor.

The production route uses the file-backed `MYOTA_GEODATA_WORK` WorkQueue with
four target durables. All four target durables are empty with one active pull
waiter each. The four prior Geodata durables remain provisioned in the shared
Interest-retained `MYOTA_EVENTS` stream, inactive and empty. Both streams have
zero messages; the target WorkQueue has finite limits. No accepted production
Geodata work was available to exercise, and no production data was changed.

The rollout audit found the digest values only triggered pod restarts while
container references still used mutable `:latest` tags. Deploy now uses
`repository@sha256:…` references wherever a production digest is configured;
Platform mirrors this source. The corrected chart was applied at Helm revision
191 and the live images now match the pins.

The 24-hour rollback observation starts from the latest completed rollout,
revision 192, at 21:39:22 UTC on 10 October and ends no earlier than
21:39:22 UTC on 11 October. Keep all four old durables until the final
topology, backlog, recovery and Fleet checks, then retire only those four. Keep
Activity's notification durable, `MYOTA_EVENTS`, PostgreSQL source/job/outbox
records, recovery columns/indexes and historical DLQs.

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

## Post-observation retirement procedure — prepared, not executed

After the 24-hour gate has elapsed, perform the final checks in this order:

1. Confirm no later Helm rollout has reset the observation clock. Require Helm
   revision 192 to remain deployed, Fleet Ready=True at the expected commit,
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
