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
  - **Complete:** Phases 0–4 within their recorded evidence bounds.
  - **Phase 5 live:** Four Geodata work kinds route to the bounded
    `MYOTA_GEODATA_WORK` stream; migration 021 and Activity idempotency fix
    are deployed. Helm 191 is deployed and Fleet is Ready=True at
    `a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109`, 60/60 resources. Configured digest refs match pod image IDs.
    The target durables are empty with active workers.
  - **Rollback:** Four legacy Geodata durables remain inactive and empty until
    the observation ends no earlier than 21:28:41 UTC on 11 October 2026. Do not remove Activity's
    notification durable or `MYOTA_EVENTS`.
  - **Still open:** The 24-hour observation and safe retirement of those four
    durables, then the separate Phase 6 fact-stream retention transition. See
    [Phase 5 evidence](evidence/phase5-geodata-work-2026-10-10.md),
    [Phase 5 plan](nats-event-migration-plan.md), and
    [Phase 4 evidence](evidence/phase4-activity-work-2026-10-10.md).
- [JetStream recovery and replay runbook]- [JetStream recovery and replay runbook](jetstream-recovery.md) — selected
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
