# Phase 6 — shared fact-stream cutover and retirement

**Date:** 11 October 2026  
**Status:** Complete. The migration succeeded in Helm revision 210.
The one-time migration flag was disabled in Helm revision 211; the current
Helm release is revision 213 with all-topology validation enabled. The MyOTA Fleet bundle is Ready=True at Deploy
main commit `16af2edf6b8e868904f9284e481d971ae7d6d38c`; every MyOTA Deployment
has all desired replicas ready.

## Decision and scope

Use one direct, coordinated cutover of the existing shared fact stream.
No shadow traffic, canary, synthetic production messages, or new metrics
baseline is required. Phases 2–5 already moved the selected producers and work
consumers to their registered JetStream routes. The cutover changes the
remaining shared-stream retention and subject capture, then leaves the
deployment-owned provisioner validating the resulting topology on upgrades.

Keep the existing Surveyor, dashboards, alerts, and operational runbooks.
They are existing operating controls, not a new metrics workstream. The
Operations service remains read-only for broker metadata.

## Preflight evidence

Before migration, read-only checks on the host's K3s deployment found:

- `MYOTA_EVENTS` had Interest retention, file storage, one replica, a 30-day
  age, unbounded byte/message/message-size limits, `DiscardOld`, and subjects
  `myota.events.>` plus the retired `myota.geodata.>` prefix.
- The stream had 0 messages and 0 bytes. `activity-notifications-v1` was its
  only durable; its 21 exact registered fact filters were present with zero
  pending, ack-pending, and redelivery counts.
- The six Activity and four Geodata target work durables existed with their
  registered filters. Both WorkQueue streams had zero messages and bytes.
- Each of the three owner-database outboxes had 0 unpublished rows (0 ready,
  0 delayed). The Geodata database retained six unresolved application
  dead-letter records: three `geodata.import.queued.v1` and three
  `geodata.import.recovered.v1`. They remain in the owning database; this
  retention-only change neither purged nor redrove them.
- The four legacy Geodata durables had already been retired under Phase 5.
  No source database, migration marker, outbox record, DLQ record, or
  idempotency state was deleted for this phase.

## Implementation and live result

The Deploy-owned `services/provision_jetstream.py` now has one explicit
`NATS_EVENTS_RETENTION_MIGRATION=1` path. It only updates the exact reviewed
legacy configuration after checking an empty stream, zero counters, and the
single expected Activity notification durable. Any other state fails closed.
Normal stream and consumer provisioning remains drift-checked; relay replicas
do not mutate topology.

The shared relay in `services/outbox_worker.py` now requires the exact
`myota.events.>` subject capture, Limits retention, file storage, one replica,
finite limits, and `DiscardNew`. `MYOTA_EVENTS` is no longer a Geodata work
route. The production Fleet values enabled the migration only for the
revision 210 pre-upgrade hook. After its successful application, the host
values were changed to `migrateEventsStream: false`; the normal
create-only/drift-validation hook remains enabled for all three streams.

The migration hook logged successful conversion and validation of
`MYOTA_EVENTS`, both WorkQueue streams, and all ten work durables. A fresh
read-only live inspection found:

| Stream | Subjects | Retention | Age | Max bytes / messages / message | State |
|---|---|---|---:|---|---|
| `MYOTA_EVENTS` | `myota.events.>` | Limits | 30 days | 1 GiB / 500,000 / 1 MiB | 0 messages, 0 bytes |
| `MYOTA_ACTIVITY_WORK` | `myota.work.activity.>` | WorkQueue | 30 days | 1 GiB / 250,000 / 1 MiB | 0 messages, 0 bytes |
| `MYOTA_GEODATA_WORK` | `myota.work.geodata.>` | WorkQueue | 30 days | 3 GiB / 500,000 / 1 MiB | 0 messages, 0 bytes |

All streams use file storage, one replica, and `DiscardNew`. The 21-filter
Activity notification durable remains active. Six Activity and four Geodata
durables retain their exact registered work filters; all report zero pending,
ack-pending, and redeliveries. Core, Activity, and Geodata outbox queues remain
at zero unpublished rows after rollout. The Postgres DLQ records remain
unchanged.

Helm revision 210 applied the cutover. Follow-up revision 211 disables the
one-time migration environment flag while keeping the all-topology
drift-validation hook enabled. The current revision 213 is deployed, and Fleet reports Ready=True at Deploy
main commit `de01650e19b88bc99a11ede1f25694202c8c8d17`.
All MyOTA Deployment desired replicas are ready.

## Isolated migration qualification and cleanup

Using the same published Deploy runtime image,
`ghcr.io/myota-platform/myota-service@sha256:a3767bc1dac6701f49537ddb15e0b1ed560ff6228e015eca577ae8dec76d9c77`,
a disposable `phase6-nats-migration` namespace hosted a local NATS server.
The test seeded an empty legacy Interest-retained stream and one pull durable,
ran the gated migration, then reran provisioning. The first run converted and
validated all three target streams and ten work durables; the second run
reported the fact-stream migration as a no-op and validated the target again.
No message was published. The namespace and its resources were deleted and
verified absent.

Deploy CI passed the focused Python quality and Helm rendering workflows on
main commit `16af2edf6b8e868904f9284e481d971ae7d6d38c`
(runs 38099669717 and 38099669405).

## Recovery and accepted limits

Interest retention had already removed previously acknowledged or unmatched
facts. A zero-message preflight proved no retained messages would be lost in
this conversion; it does not restore historical events. PostgreSQL domain
state, owner work rows, outboxes, database DLQs, and consumer idempotency
remain authoritative. Facts are now retained for the bounded 30-day window.
Replay after that window requires a service-owned history source or rebuilding
from current authoritative state; JetStream is not an event archive.

Do not roll the stream back to Interest or reintroduce `myota.geodata.>` into
the fact stream. Recovery is code/workload rollback while preserving the
Limits fact stream and current WorkQueue streams; accepted work is recovered
from owner rows/outboxes using stable IDs. Off-node broker recovery remains
deferred by the project owner. The local disposable-PVC recovery qualification
and [JetStream recovery runbook](../jetstream-recovery.md) remain the supported
scope.

## Phase 6 exit review

- [x] All registry-approved producers publish through the database-specific
  outbox relays and registered fact/work subjects. The only selected fact
  consumer remains Activity notifications; work consumers use their ten exact
  WorkQueue durables. Operations remains read-only.
- [x] The mixed Interest stream is gone; the fact stream has the approved bounded
  Limits policy, and no Geodata work subject is captured by it.
- [x] Live relay/outbox state, consumer filters/counters, target capacity settings,
  Helm hook results, and isolated migration/idempotency behavior are verified.
- [x] Existing NATS Surveyor dashboards, alerts, recovery docs, and redrive
  ownership remain in operation. No new metrics baseline, shadow traffic, or
  canary rollout was run or added.
- [x] The current Helm revision 213 is deployed with its one-time migration flag
  disabled and all-topology drift validation enabled; every MyOTA Deployment is ready,
  and the MyOTA Fleet bundle is Ready=True at Deploy commit
  `de01650e19b88bc99a11ede1f25694202c8c8d17`.
