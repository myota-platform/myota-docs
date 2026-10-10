# JetStream recovery and replay runbook

**Status:** Selected recovery procedure; production qualification is incomplete.
Do not use these steps to restore into the production broker until the recovery
gate in the NATS migration plan is closed.

## Recovery authority

PostgreSQL in each owning service remains authoritative for accepted facts and
work. JetStream is a bounded transport and replay window, not the event archive.
Keep stream snapshots off the NATS node and keep topology and credential
configuration in their separately managed source. Encrypt backups and restrict
access to the same data classification as the messages they contain.

Operations is limited to read-only stream and consumer metadata. It must not
fetch message payloads, consume, acknowledge, create, update, delete, purge, or
restore streams. Use a separately authorized deployment or recovery identity
for administrative actions.

## Before a backup

1. Confirm broker health, free PVC space, stream names, storage and retention
   settings, limits, stream sequence range, and each durable's acknowledged and
   delivered sequence.
2. Confirm no migration or relay cutover is in progress. For the legacy shared
   MYOTA_EVENTS stream, record its Interest retention and current consumers;
   do not change its retention as part of backup.
3. Use the approved JetStream stream-backup mechanism to capture each target
   stream and its durable state. Record tool and server versions, UTC start and
   finish times, stream metadata, and backup checksum.
4. Copy the completed backup to storage outside the Kubernetes node and verify
   the copy by checksum. Store the stream snapshot separately from the
   declarative topology and out-of-band credential recovery procedure.
5. Record the source database watermark or outbox event/work IDs used for
   reconciliation. Never include database credentials or NATS private seeds in
   the backup record.

A PVC snapshot by itself is not an off-node backup and does not close the
recovery gate.

## Restore qualification

Restore into an isolated namespace and a fresh disposable broker with storage
separate from production. Never point restored consumers at production
databases or external side-effecting services.

1. Restore the selected stream snapshot and verify stream name, subjects,
   retention policy, storage type, limits, replica count, first/last sequence,
   message count, and byte count.
2. Run the deploy-owned create-only provisioner with the reviewed topology.
   Confirm it validates existing configuration and recreates missing durables;
   any configuration drift must fail closed.
3. Compare a sample of restored message IDs and sequence positions with the
   owning service's source outbox records. Compare durable ACK floors and
   pending counts with the captured backup evidence.
4. For fact replay, create a new isolated validation durable with an explicit
   start sequence or time. Check schema/version compatibility and duplicate
   handling without invoking production side effects. Delete only the
   disposable test durable after recording results.
5. For work, inspect the owning database's job state and consumer idempotency
   checkpoint. Re-drive from that database with the original stable work ID;
   do not reconstruct accepted work from broker contents alone.
6. Record duplicate-delivery behavior, message and durable comparisons,
   application checkpoints, tool/server versions, test data IDs, and cleanup
   evidence. Remove the isolated namespace and temporary backup copy after the
   evidence has been retained in the approved off-node location.

## Outage, capacity, and expiry handling

All target streams use finite limits and DiscardNew. At capacity, a publish
must fail and remain recoverable in its owning database/outbox; do not delete
old messages or switch to DiscardOld. Page the operator when a stream
approaches its byte, message, age, or PVC threshold. Work that approaches the
30-day maximum age must be reconciled and re-driven from its owning database
using the same work ID before expiry.

During an outage, leave accepted source rows pending and let the relay retry
through its bounded retry/dead-letter policy. Before replay or re-drive, inspect
the relevant database row and idempotency checkpoint. Do not rewind a
production side-effecting durable.

Under the current Interest-retained legacy stream, messages already removed
after all interested durables acknowledged them cannot be recovered from a
snapshot taken later. At cutover, stop relays, inspect and snapshot the current
stream, reconcile remaining events against database outboxes, and preserve the
legacy stream until its consumer and replay needs are resolved.

## Qualification evidence and remaining work

The 10 October 2026 disposable-broker drill restored a stream and its durable
ACK state, then created a new durable and replayed three synthetic messages.
The test namespace and its temporary files were removed. This proves a bounded
restore/replay mechanic only.

Still required before Phase 1 exit:

- Create and checksum an off-node backup using the production backup
  destination and credential path.
- Qualify disposable PVC loss, off-node restore, durable recreation, duplicate
  delivery, disk-capacity rejection, relay retry/dead-letter, and bounded
  replay.
- Compare restored IDs and ACK state with real source outbox watermarks.
- Record alarms and operator actions for capacity pressure and work redrive.
- Review the run with the workspace owner and retain the evidence without
  publishing sensitive message contents or credentials.

See the [joint Phase 1 review](evidence/phase1-joint-review-2026-10-10.md) and
the [migration plan](nats-event-migration-plan.md) for the selected topology and
remaining gates.
