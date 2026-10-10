# Messaging and NATS

- [JetStream administration and status](jetstream-admin-status.md) — read-only
  broker state and sampled history.
- [NATS event migration plan](nats-event-migration-plan.md) — Phase 0 complete;
  Phase 1 contract and topology implementation in progress. The delegated review
  decided dispositions for all 68 fact schemas and selected credential,
  capacity, recovery, and provisioning policies; production gates remain open.
- [JetStream recovery and replay runbook](jetstream-recovery.md) — selected
  PostgreSQL recovery authority, isolated restore/replay procedure, and remaining
  qualification evidence.
- [Phase 0 event and work inventory](nats-event-migration-inventory.md) —
  repository evidence, current queues, replay limits, mirror status, and
  selected topology. The decision is recorded in
  [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
- [Joint review](evidence/phase1-joint-review-2026-10-10.md), [Phase 1 evidence](evidence/phase1-contract-topology-2026-10-09.md), and
  [current/target topology diagrams](../../architecture/diagrams/nats-event-migration.md).
