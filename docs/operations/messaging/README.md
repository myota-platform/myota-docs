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
  - **Complete:** Phases 0–4 within their recorded evidence bounds. Phase 4
    Activity work cutover and schema retirement are complete.
  - **Phase 5 live:** Four Geodata work kinds route to `MYOTA_GEODATA_WORK`;
    migration 021 and the retry-safe partial-deletion recovery fix are deployed.
    Four replacement durables have exact filters and zero backlog. No accepted
    production Geodata work was available at cutover.
  - **Rollback:** Four legacy Geodata durable definitions remain empty and
    inactive until the 24-hour observation expires at 20:55:08 UTC on 11 October
    2026. Do not remove Activity's notification durable or `MYOTA_EVENTS`.
  - **Still open:** Fleet Ready reconciliation, two-database cross-service
    failure qualification, cancellation/expiry replay chains, final legacy
    durable retirement, and Phase 6 fact-stream retention transition. See the
    [Phase 5 evidence](evidence/phase5-geodata-work-2026-10-10.md), [Phase 5
    plan](nats-event-migration-plan.md), and [Phase 4 evidence](evidence/phase4-activity-work-2026-10-10.md).
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
