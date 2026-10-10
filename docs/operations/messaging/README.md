# Messaging and NATS

- [JetStream administration and status](jetstream-admin-status.md) — read-only
  broker state and sampled history.
- [NATS event migration plan](nats-event-migration-plan.md)
  - **Complete:** Phase 0 inventory and Phase 1 contract/topology work. The
    delegated review dispositioned all 68 fact schemas.
  - **Selected:** Cluster-internal trust boundary without NATS auth/TLS, bounded
    work schemas, create-only provisioner, and accepted capacity limits.
  - **Verified:** Fail-closed opt-in Helm pre-upgrade gate and local
    restore/replay evidence.
  - **Still open:** Runtime path changes and production cutover in Phase 2.
- [JetStream recovery and replay runbook](jetstream-recovery.md) — selected
  PostgreSQL recovery authority, isolated restore/replay procedure, and remaining
  qualification evidence.
- [Phase 0 event and work inventory](nats-event-migration-inventory.md) —
  repository evidence, current queues, replay limits, mirror status, and
  selected topology. The decision is recorded in
  [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
- [Joint review](evidence/phase1-joint-review-2026-10-10.md), [Phase 1 completion evidence](evidence/phase1-completion-2026-10-10.md), [historical 9 October evidence](evidence/phase1-contract-topology-2026-10-09.md), and
  [current/target topology diagrams](../../architecture/diagrams/nats-event-migration.md).
