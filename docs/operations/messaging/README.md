# Messaging and NATS

- [Activity domain-event notification consumer](activity-notification-consumer.md)
  — exact registered filters, durable rollout, retries, metrics, and audited
  poison-event redrive.
- [Activity JetStream work queues](activity-work-queues.md) — six job kinds,
  registered subjects/durables, lease and retry behavior, dead-letter recovery,
  schema retirement, and staged rollout checks.
- [JetStream administration and status](jetstream-admin-status.md) — read-only
  broker state and sampled history.
- [NATS event migration plan](nats-event-migration-plan.md)
  - **Complete:** Phases 0–3 within their recorded evidence bounds. Phase 4
    code and isolated PostgreSQL/JetStream qualification are complete. Its
    production cutover is not complete: the live Activity worker still polls
    PostgreSQL, the live target Activity stream is absent, and the current
    Activity schema still has its old claim index. The exact staged gate and
    results are in the [Phase 4 evidence](evidence/phase4-activity-work-2026-10-10.md)
    and [work queue runbook](activity-work-queues.md).
  - **Selected:** Cluster-internal trust boundary without NATS auth/TLS, bounded
    work schemas, create-only provisioner, and accepted capacity limits.
  - **Verified:** Fail-closed Helm pre-upgrade gate, local restore/replay, and
    Phase 4 disposable broker/database migration and duplicate-safe ACK checks.
  - **Still open:** Activity production worker drain/migration/rollout, Activity
    compatibility-repair removal, payload/privacy enforcement, Geodata work
    migration, and controlled stream-retention cutover. The live shared stream
    remains Interest-retained. Helm revision 173 is deployed and Fleet reports
    the MyOTA bundle Ready; see live state in the Phase 4 evidence.
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
