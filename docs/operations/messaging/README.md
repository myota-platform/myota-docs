# Messaging and NATS

- [Activity domain-event notification consumer](activity-notification-consumer.md)
  — exact registered filters, durable rollout, retries, metrics, and audited
  poison-event redrive.
- [Activity JetStream work queues](activity-work-queues.md) — six job kinds,
  registered subjects/durables, lease and retry behavior, dead-letter recovery,
  schema retirement, and staged rollout checks.
- [NATS Surveyor monitoring consolidation](../../observability/nats-surveyor-migration.md)
  — central broker metrics and Grafana dashboard inventory; tracks live image rollout and the Operations history-table migration.
- [JetStream administration history](jetstream-admin-status.md) — retired Admin page behavior and cutover evidence.
- [NATS event migration plan](nats-event-migration-plan.md)
  — Phases 0–6 are complete within their documented evidence bounds.
  Helm revision 211 is deployed; MyOTA Fleet is Ready=True at Deploy commit
  `16af2edf6b8e868904f9284e481d971ae7d6d38c`. `MYOTA_EVENTS` uses bounded
  Limits retention and captures only registered facts; Activity and Geodata
  work remain on separate WorkQueue streams. The migration flag is disabled
  and topology drift validation remains enabled. The user explicitly waived
  Phase 5's 24-hour observation; it closed early, not as a full-day observation.
  See the [Phase 6 evidence](evidence/phase6-fact-stream-cutover-2026-10-11.md),
  [Phase 5 evidence](evidence/phase5-geodata-work-2026-10-10.md), and
  [NATS monitoring plan](../../observability/nats-surveyor-migration.md).
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
