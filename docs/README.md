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
- [Structured logging implementation](observability/logging-implementation.md) — current phased status, Phase 7 receiver gate, and evidence.
- [MyOTA observability overview](observability/overview.md) — metrics, traces, logs, dashboards, and current live gaps.
- [NATS migration plan](operations/messaging/nats-event-migration-plan.md) —
  Phases 0–6 are complete within their documented evidence bounds. Helm
  revision 213 is deployed; MyOTA Fleet is Ready=True at Deploy main commit
  `dfc423d26bd79a82b1177b9e5d51e2d1196580c9`. `MYOTA_EVENTS` now has
  bounded Limits retention for registered facts, while Activity and Geodata
  work remain on separate WorkQueue streams. See the
  [Phase 6 evidence](operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md),
  [Phase 5 evidence](operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
  and [ADR-0008](architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
- [NATS Surveyor monitoring consolidation](observability/nats-surveyor-migration.md) —
  live Surveyor, dashboard, alert and bounded capacity decisions. Fleet is Ready
  on Deploy main at Helm revision 208; the replacement images and forward-only
  Operations history migration are deployed. The retired combined dashboard
  and its two PostGIS panels are not recreated.
- [Observability](observability/README.md) — telemetry architecture, logging
  roadmap, Surveyor migration plan and current verification.
- [Structured logging implementation status](observability/logging-implementation.md) —
  Phase 1–5 source/configuration plus a live Identity log/trace and all seven
  dashboard query checks. Authenticated UI navigation and Phase 6–7 gates remain open; Fleet is
  Ready=True.
- [Logging implementation evidence](observability/evidence/logging-implementation-2026-10-11.md) —
  CI, K3s, runtime image and end-to-end acceptance evidence.
- [Logging operations runbook](observability/logging-runbook.md) — current LogQL
  queries and troubleshooting guidance.
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
