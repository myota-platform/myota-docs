# Messaging and NATS

- [Activity domain-event notification consumer](activity-notification-consumer.md)
  — exact registered filters, durable rollout, retries, metrics, and audited
  poison-event redrive.
- [Activity JetStream work queues](activity-work-queues.md) — six job kinds,
  registered subjects/durables, lease and retry behavior, dead-letter recovery,
  schema retirement, and staged rollout checks.
- [NATS Surveyor monitoring consolidation](../../observability/nats-surveyor-migration.md)
  — proposed central broker metrics and Grafana dashboard; includes staged
  removal of the current Admin page, pollers and database history.
- [Current legacy JetStream administration page](jetstream-admin-status.md) — read-only broker state and sampled history, pending the planned Surveyor cutover.
- [NATS event migration plan](nats-event-migration-plan.md)
  - **Complete:** Phases 0–5 within their documented evidence bounds. Helm
    revision 193 is deployed; Fleet Ready=True at Deploy commit
    cfecd655d9c0eee9d19db26725fb11c99366815a.
  - **Phase 5:** The four legacy Geodata durables were retired after the final
    filter, backlog, migration, owner-row recovery-age and Fleet checks. The
    user waived the 24-hour elapsed-time requirement; closure was early and
    is not reported as a full 24-hour observation. Activity and target
    durables, migration 021, work/outbox state and the Interest-retained
    MYOTA_EVENTS stream remain.
  - **Next:** Phase 6 fact-stream retention work. NATS monitoring consolidation
    is a separate planned observability item. See [Phase 5 evidence](evidence/phase5-geodata-work-2026-10-10.md)
    and [the monitoring plan](../../observability/nats-surveyor-migration.md).
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
