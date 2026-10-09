# ADR-0008: NATS JetStream event and work topology

## Status

Accepted for the target design by the workspace owner. Phase 0 inventory,
decision, and evidence-owner assignments are complete. Runtime implementation,
evidence closure, and production qualification remain open.

## Context

The current deployment publishes database outbox rows through three instances
of one relay into a shared, file-backed `MYOTA_EVENTS` stream. That stream
captures both `myota.events.>` and `myota.geodata.>`, currently uses Interest
retention, and provisions one Activity notification durable plus four Geodata
work durables. Activity also accepts asynchronous jobs from PostgreSQL and
polls them with a worker. Interest retention makes consumer provisioning part
of publish correctness, while the current broad Activity durable acknowledges
many events as no-ops. JetStream is not the authoritative store for domain
state or a permanent event archive.

The source scan and row-level evidence are in the
[Phase 0 inventory](../../operations/messaging/nats-event-migration-inventory.md).
This decision selects the target design; it does not claim any runtime change
or operational qualification is complete.

## Decision

### Streams and subjects

Create three streams with disjoint subject namespaces:

| Stream | Subject capture | Retention | Purpose |
|---|---|---|---|
| `MYOTA_EVENTS` | `myota.events.>` | Limits | Bounded retention window for committed domain facts and independent event consumer groups |
| `MYOTA_ACTIVITY_WORK` | `myota.work.activity.>` | WorkQueue | Competing Activity worker commands |
| `MYOTA_GEODATA_WORK` | `myota.work.geodata.>` | WorkQueue | Competing Geodata worker commands |

Use file storage. Set finite `MaxAge`, `MaxBytes`, `MaxMsgs`, and
`MaxMsgSize` on all streams, with `DiscardNew` so capacity pressure rejects
new publishes and allows database outbox relays to apply backpressure. Select
numeric limits from measured traffic, allocated storage, maximum job duration,
and recovery service objectives before implementation. Do not leave limits
unbounded.

Facts use `myota.events.<eventType>`, preserving the dotted event type and
version suffix. For example,
`identity.account.created.v1` maps to
`myota.events.identity.account.created.v1`. Do not encode dots as underscores.
Maintain a checked-in event registry and per-event JSON Schemas in the
authoritative `myota-contracts` repository.

The work namespaces are disjoint from both the target fact stream and the
current stream's `myota.geodata.>` capture. This avoids a subject being
captured by two streams during rollout. WorkQueue filters must be disjoint
within each stream; provision one durable per work kind and share that durable
among replicas of the same worker group.

### Event and work disposition

- Keep committed domain facts on `MYOTA_EVENTS`. Create one durable per
  independent consumer group, with explicit filters for facts that group
  handles. Activity notifications use scoped filters for Identity facts and
  the two Geodata review/status facts it processes. Remove the broad no-op
  subscription after migration.
- Move Activity's six actual asynchronous work kinds to
  `MYOTA_ACTIVITY_WORK`: QSO ingestion, ADIF import, award recalculation,
  award evaluation, PDF rendering, and statistics rebuild.
- Do not migrate the synthetic Activity `NOTIFICATION_SEND` job. Its current
  handler only changes a database status and does not contact a delivery
  provider. Correct or remove that state-only job in a later Activity
  implementation. If external notification delivery is introduced, model it
  as a separate provider-backed command with its own idempotency contract.
- Move Geodata preprocessing, import promotion, confirmed deletion, and
  location enrichment to `MYOTA_GEODATA_WORK`.
- Keep scheduled retention cleanup and database reconciliation as scheduled
  maintenance/recovery. They are not normal broker work. Keep startup recovery
  and lease-expiry scans as recovery mechanisms that recreate or resume work
  from the owning database.
- Keep Operations read-only for JetStream inspection. It must not publish
  business events, consume/ack messages, or mutate stream/consumer
  configuration.

### Envelope, delivery, and recovery

Use an immutable versioned envelope with `envelopeVersion: 1`, event/work
identity, type, occurrence time, producer, aggregate identity where relevant,
trusted correlation context, optional causation identity, and payload. Keep the
event contract version in `eventType` (for example, `.v1`) separate from the
envelope version. Do not publish a mutable relay-attempt count in the envelope;
keep retry data in outbox, consumer, and dead-letter records.

Delivery is at least once. Publish with a stable `Nats-Msg-Id` derived from
`eventId` or `workId`, but use database uniqueness and committed
idempotency/checkpoint state as the correctness boundary. NATS duplicate
suppression is limited to a configured time window and is not exactly-once
delivery. Ack only after database side effects and idempotency/checkpoint state
commit.

On terminal consumer failure, persist an idempotent application dead-letter
record before terminating the message, alert on both stored failures and
JetStream max-delivery advisories, and provide an authorized, audited redrive
path. JetStream does not create or populate a dead-letter queue automatically.
For work redrive, re-enqueue from the owning database with the same work
identity. For fact replay, use a separate durable with explicit start
sequence/time and a reviewed side-effect policy; do not rewind a production
consumer. After the bounded fact retention period, rebuild projections from
source-of-truth state. If historical facts are required, keep them in a
service-owned history store.

Keep PostgreSQL domain state, outbox/dead-letter rows, and consumer
idempotency/checkpoint records authoritative. Ensure recovery can recreate
unacknowledged work before its WorkQueue retention limit expires.

### Deployment and migration

Keep JetStream replica count at one while the deployed NATS topology is a
single server. This is persistent single-node operation, not high availability;
require tested off-host backup and restore. Set stream replicas to three only
after deploying and validating a three-server JetStream cluster. Replication
protects against node loss but adds storage and write load and does not improve
write throughput.

Use one controlled, drift-checked provisioner for stream and durable
configuration; relay replicas must not race to mutate broker topology. Stop
relays and inspect stream state before changing the current
`MYOTA_EVENTS` retention from Interest to Limits because the transition
applies immediately and cannot restore messages already removed. Create the two
WorkQueue streams on their new disjoint subjects; WorkQueue is not a live
retention conversion. Provision durables before routing producers. Move pending
legacy Geodata outbox rows through one publish path without dual-publishing.
Drain legacy messages and consumers only after rollback and database recovery
checks pass.

## Consequences

- Domain facts and competing work have separate retention semantics, storage
  caps, dashboards, and failure blast radii.
- The fact stream can support bounded independent fan-out without claiming to
  be a permanent event archive.
- WorkQueue removes work after the first successful ack; each work kind needs
  a single competing durable with a non-overlapping filter.
- Finite limits and `DiscardNew` make capacity exhaustion visible as publish
  backpressure. Operators must alert and restore capacity before outbox
  backlogs exceed database recovery limits.
- A separate provisioner, schemas/registry, DLQ workflow, replay tooling, and
  recovery evidence add implementation work and are Phase 1+ gates.
- Current single-node deployment remains a single point of failure until the
  NATS cluster is deployed and qualified.

## Evidence and references

- [Phase 0 event and work inventory](../../operations/messaging/nats-event-migration-inventory.md)
- [NATS retention policies](https://docs.nats.io/learn/jetstream/retention-policies)
- [NATS stream subjects and setup](https://docs.nats.io/learn/jetstream/your-first-stream)
- [NATS publishing and deduplication](https://docs.nats.io/learn/jetstream/publishing)
- [NATS acknowledgments and redelivery](https://docs.nats.io/learn/jetstream/acknowledgment)
- [NATS stream capacity and discard policies](https://docs.nats.io/learn/jetstream/shaping-the-stream)
- [NATS node-loss resilience and replication](https://docs.nats.io/learn/jetstream/surviving-node-loss)
