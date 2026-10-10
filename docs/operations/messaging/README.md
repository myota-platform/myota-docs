# Messaging and NATS

- [JetStream administration and status](jetstream-admin-status.md) — read-only
  broker state and sampled history.
- [NATS event migration plan](nats-event-migration-plan.md) — Phase 0 complete;
  Phase 1 contract/provisioning preparation in progress. Source-derived payload
  schemas cover Identity, Programme, Activity, and Geodata and pass CI; owner/privacy
  review, Geodata payload minimization, and later migration phases remain open.
- [Phase 0 event and work inventory](nats-event-migration-inventory.md) —
  repository evidence, current queues, replay limits, mirror status, and
  selected topology. The decision is recorded in
  [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
- [Phase 1 evidence](evidence/phase1-contract-topology-2026-10-09.md) and
  [current/target topology diagrams](../../architecture/diagrams/nats-event-migration.md).
