# JetStream recovery and replay runbook

**Status:** Local disposable-PVC recovery and replay are qualified for the
single-node scope. Off-node recovery is explicitly deferred by the project
owner; do not treat this runbook as a production off-node backup procedure.

## Recovery authority

PostgreSQL in each owning service remains authoritative for accepted facts and
work. JetStream is a bounded transport and replay window, not the event archive.
Keep topology and credential configuration in their separately managed source.
Any future off-node snapshots must be encrypted and restricted to the same
data classification as the messages they contain. Off-node backup is not a
current Phase 1 gate.

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
3. For local qualification, use a disposable PVC snapshot/copy and record tool
   and server versions, UTC start and finish times, stream metadata, and
   checksums. The 10 October drill completed this step.
4. A separate off-node backup destination and credential path may be added in
   a future recovery phase; this is explicitly deferred for the current
   single-node deployment.
5. Record the source database watermark or outbox event/work IDs used for
   reconciliation. Never include database credentials or NATS private seeds in
   the backup record.

The qualified local PVC copy protects only against broker-PVC replacement on
the same node. Node or cluster loss can remove the bounded transport window.

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
   evidence has been recorded without retaining test payloads or credentials.

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

## Outbox dead-letter inspection and redrive

The relay stores terminal failures in the owning database. Use the CLI inside
the matching relay deployment; do not copy event payloads out of PostgreSQL.
Restrict Kubernetes `pods/exec` permission on relay deployments to operators
authorized to perform redrive. Kubernetes API audit logs are the identity
evidence; the CLI's `--actor` is a recorded label and is not independently
authenticated.

```sh
k3s kubectl -n myota exec deploy/myota-core-outbox -- \
  python3 services/outbox_admin.py list --limit 100

EVENT_UUID="replace-with-event-uuid"
OPERATOR_ID="replace-with-authenticated-operator-name"
REASON="cause corrected and source row verified"
k3s kubectl -n myota exec deploy/myota-core-outbox -- \
  python3 services/outbox_admin.py redrive "$EVENT_UUID" \
  --actor "$OPERATOR_ID" \
  --reason "$REASON"
```

Use `myota-activity-outbox` or `myota-geo-outbox` for their respective
databases. Confirm the event type and redacted error, correct the cause, verify
the retained source row and consumer idempotency behavior, then redrive. The
CLI resets the owning outbox row with the original event ID and records actor,
reason, prior attempt count, and prior error in `outbox_redrive_audit`. A later
successful publish resolves the dead-letter entry. Never redrive an unresolved
failure repeatedly without investigating it.

Under the current Interest-retained legacy stream, messages already removed
after all interested durables acknowledged them cannot be recovered from a
snapshot taken later. At cutover, stop relays, inspect and snapshot the current
stream, reconcile remaining events against database outboxes, and preserve the
legacy stream until its consumer and replay needs are resolved.

## Qualification evidence and remaining work

The 10 October 2026 disposable-broker drill restored a stream and its durable
ACK state from a replacement local PVC, then created a new durable and replayed
three synthetic messages. It also verified create/idempotency, drift rejection,
and `DiscardNew` capacity rejection. The test namespace and temporary storage
were removed. This qualifies only single-node local recovery mechanics.

Phase 1 exit criteria are complete within the accepted single-node scope.
Deferred work and limits:

- Off-node snapshot/restore remains deferred. If the project later requires
  node-loss recovery, add an approved backup destination and qualify its restore.
- Production outbox watermark comparison and compatibility rollout remain
  open. Relay retry, dead-letter, and same-ID crash recovery passed in the
  isolated Phase 2 drill; see the
  [Phase 2 evidence](evidence/phase2-relay-hardening-2026-10-10.md).
- Retain alarms and operator actions for capacity pressure and work redrive as
  operational follow-up; PostgreSQL remains authoritative for reconciliation.

See the [joint Phase 1 review](evidence/phase1-joint-review-2026-10-10.md) and
the [migration plan](nats-event-migration-plan.md) for the selected topology and
remaining gates.
