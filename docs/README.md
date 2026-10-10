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
  Phases 0–5 are complete within their documented evidence bounds. Phase 5's
  four legacy Geodata durables were retired after final checks under an
  explicit early-close waiver; no claim is made that 24 hours elapsed. Helm
  revision 193 remains deployed and Fleet reports Ready=True at
  cfecd655d9c0eee9d19db26725fb11c99366815a. See [Phase 5 evidence](operations/messaging/evidence/phase5-geodata-work-2026-10-10.md)
  and [Phase 6 plan](operations/messaging/nats-event-migration-plan.md#phase-6).
- [NATS Surveyor monitoring consolidation](observability/nats-surveyor-migration.md) —
  proposed Surveyor deployment, dashboard 16256 compatibility, seven-day
  overlap, and staged removal of the current Admin page, broker pollers,
  Grafana dashboard and NATS snapshot table.
- [Observability](observability/README.md) — telemetry architecture, logging
  roadmap, Surveyor migration plan, prompts, and recent verification.
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
