# MyOTA documentation index

The repository-root [README](../README.md) is the short project overview. This
index organizes the detailed documentation by subject; the dedicated
[To do](to-do/README.md) and [Work in progress](work-in-progress/README.md)
indexes organize open work by delivery status.

## Browse by subject

- [Architecture and decisions](architecture/README.md) — system boundaries,
  repository ownership, ADRs, and diagrams.
- [Domain, API, and client contracts](domain/README.md) — API migration,
  programmes, identity, administration, and awards.
- [Geodata](geodata/README.md) — entity lifecycle, scaling, upload, and query
  evidence, plus implementation prompts.
- [Geodata Phase 5 live evidence](geodata/evidence/phase5-staged-rollout-2026-10-09.md)
  — staged K3s rollout, HPA up/down, load comparison, cleanup, and open limits.
- [Operations](operations/README.md) — runbooks, production setup, NATS, and
  storage administration.
- [NATS migration plan](operations/messaging/nats-event-migration-plan.md) —
  Phases 0–4 are complete within their recorded evidence bounds. Phase 4
  deployed the Activity WorkQueue, six exact durables, migrated queued work,
  retired database polling and its claim index, and disabled compatibility
  repair. Production had no selected jobs at cutover; all six work types passed
  isolated PostgreSQL/JetStream processing qualification. See the [Phase 4 evidence](operations/messaging/evidence/phase4-activity-work-2026-10-10.md),
  [Activity work queues runbook](operations/messaging/activity-work-queues.md),
  [Phase 3 evidence](operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
  [migration inventory](operations/messaging/nats-event-migration-inventory.md),
  [current/target diagrams](architecture/diagrams/nats-event-migration.md), and
  [ADR-0008](architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
  The live `MYOTA_EVENTS` stream remains Interest-retained; Geodata work is
  unchanged.
- [Observability](observability/README.md) — telemetry architecture, logging
  roadmap, prompts, and recent verification.
- [Governance](governance/README.md) — charter, gap analysis, and MPOTA
  migration context.
- [History and status reconciliation](history/README.md) — project timeline,
  source inspection, and documentation reconciliation.
- [Platform policies](platform/README.md) — cross-service UTC and time handling.
- [Security](security/README.md) — threat model and identity security.
- [Development](development/README.md) — Python quality and local hooks.
- [Architecture decision records](architecture/decisions/README.md)
- [Architecture diagrams](architecture/diagrams/README.md)

## Track work by status

- [To do](to-do/README.md) — accepted backlog and proposed work that has not
  started or still needs a decision.
- [Prioritized backlog](to-do/prioritized-backlog.md) — documentation-based
  urgency order with links to the owning roadmaps.
- [Work in progress](work-in-progress/README.md) — active delivery threads,
  verification evidence, and open gates.

Status in plans and evidence uses `[x]` for work implemented or evidence-backed
as stated and `[ ]` for work not implemented, not verified, or an explicit
exit criterion still open. A completed implementation checkbox does not imply
production capacity or launch readiness; read the linked evidence and limits.

Some previous top-level `docs/*.md` paths remain as brief compatibility pages
for links published by other MyOTA repositories. The subject indexes above link
to the canonical documents.
