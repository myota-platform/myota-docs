# Messaging and NATS

- [Activity domain-event notification consumer](activity-notification-consumer.md)
  — exact registered filters, durable rollout, retries, metrics, and audited
  poison-event redrive.
- [JetStream administration and status](jetstream-admin-status.md) — read-only
  broker state and sampled history.
- [NATS event migration plan](nats-event-migration-plan.md)
  - **Complete:** Phases 0–3 within their evidence bounds. Phase 3 deployed
    Activity's exact 21-subject notification durable, transactional
    idempotency, audited poison recovery, and bounded metrics. See the
    [Phase 3 evidence](evidence/phase3-domain-consumers-2026-10-10.md) and
    [Activity notification runbook](activity-notification-consumer.md).
  - **Selected:** Cluster-internal trust boundary without NATS auth/TLS, bounded
    work schemas, create-only provisioner, and accepted capacity limits.
  - **Verified:** Fail-closed opt-in Helm pre-upgrade gate and local
    restore/replay evidence.
  - **Still open:** Payload/privacy enforcement, Activity and Geodata work
    migration, and controlled production cutover. The live stream remains
    Interest-retained. Fleet reports 60/60 resources ready, but its bundle is
    `WaitApplied` after recovery; reconcile before Phase 4 runtime changes.
- [JetStream recovery and replay runbook](jetstream-recovery.md) — selected
  PostgreSQL recovery authority, isolated restore/replay procedure, and remaining
  qualification evidence.
- [Phase 0 event and work inventory](nats-event-migration-inventory.md) —
  repository evidence, current queues, replay limits, mirror status, and
  selected topology. The decision is recorded in
  [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
- [Joint review](evidence/phase1-joint-review-2026-10-10.md), [Phase 1 completion evidence](evidence/phase1-completion-2026-10-10.md), [historical 9 October evidence](evidence/phase1-contract-topology-2026-10-09.md), and
  [current/target topology diagrams](../../architecture/diagrams/nats-event-migration.md).
- [Phase 2 relay hardening evidence](evidence/phase2-relay-hardening-2026-10-10.md)
  records source checks, isolated broker/database results, and remaining
  rollout gates.
