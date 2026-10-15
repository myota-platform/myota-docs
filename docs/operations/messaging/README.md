# Messaging and NATS

The [current system state](../../architecture/current-state.md) records the
deployed streams, consumers, Helm/Fleet revision, and accepted evidence limits.
The migration plan and phase records hold detailed decisions and verification.

## Decisions and topology

- [NATS event migration plan](nats-event-migration-plan.md) — completed Phases
  0–6, target/current topology, gates, and retained limitations.
- [Event and work inventory](nats-event-migration-inventory.md) — source-linked
  facts, work commands, consumers, retry boundaries, and replay needs.
- [JetStream recovery and replay](jetstream-recovery.md) — PostgreSQL recovery
  authority, restore/replay procedure, and qualification limits.
- [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md)
  — accepted stream and retention decisions.

## Service runbooks

- [Asynchronous email delivery](email-delivery.md) — accepted request path,
  implementation status, privacy rules, and operator procedures.
- [Activity notification consumer](activity-notification-consumer.md) — exact
  fact filters, deduplication, retries, and audited redrive.
- [Activity work queues](activity-work-queues.md) — six durable job kinds,
  database leases, retries, and recovery.
- [JetStream administration history](jetstream-admin-status.md) — retired
  Admin inspection page and cutover history.

## Evidence

- [Issue closeout](evidence/nats-readiness-issue-closeout-2026-10-11.md)
- [Phase 6 fact-stream cutover](evidence/phase6-fact-stream-cutover-2026-10-11.md)
- [Phase 5 Geodata work migration](evidence/phase5-geodata-work-2026-10-10.md)
- [Phase 1 joint review](evidence/phase1-joint-review-2026-10-10.md) and
  [Phase 1 completion](evidence/phase1-completion-2026-10-10.md).
- [Phases 2–6 evidence](nats-event-migration-plan.md) — each phase links to
  its completion record and remaining limits.
- [NATS monitoring and Surveyor](../../observability/nats-surveyor-migration.md)
