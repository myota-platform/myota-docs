# To do

This index lists accepted backlog and proposed work that has not started, or
that still needs a decision. Check the linked roadmap for item-level status and
evidence; this page is a navigation index, not a second source of truth.

For a recommended execution order and the reasons behind it, see the
[prioritized backlog](prioritized-backlog.md). Priority is based on the current
documentation evidence and does not set delivery dates.

- [Future award-condition model](../domain/awards/future-condition-model-roadmap.md)
  — define and implement typed POTA-like, worked-entity, Maidenhead, geographic,
  and diversity awards.
- [NATS event migration](../operations/messaging/nats-event-migration-plan.md)
  — Phases 0–5 are complete within their recorded evidence bounds. Helm
  revision 193 remains deployed; Fleet is Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a. The four legacy Geodata durables
  were retired after final checks. The 24-hour elapsed-time requirement was
  explicitly waived, so this is early closure rather than a completed 24-hour
  observation. Activity and target durables, migration 021, recovery data and
  the Interest-retained MYOTA_EVENTS stream remain. Phase 6 fact-stream
  retention work is next. See [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md).
- [NATS Surveyor monitoring consolidation](../observability/nats-surveyor-migration.md)
  — planned, not deployed. Deploy NATS Surveyor and a compatible Grafana
  dashboard, validate them through a seven-day overlap, then retire the
  existing Admin NATS page, duplicate broker pollers, old NATS Grafana
  dashboard, sampler alerts, Operations snapshot API/history, and its database
  table/index. Preserve outbox/worker signals, SeaweedFS operations, and
  PostGIS query panels. See the roadmap for owners and gates.
- [REST API alias retirement](../domain/api/rest-consolidation-plan.md) —
  Phase 5 remains planned until usage is measured and clients migrate.
- [Programme configuration gaps](../domain/programmes/configuration-gap-analysis.md)
  — implementation-pending configuration catalogue.
- [Admin UI internationalization](../domain/administration/i18n-roadmap.md) —
  proposed locale foundation, phased translation of admin workflows, and
  release-quality gates.
- [Charter delivery gaps](../governance/charter-gap-analysis.md) — longer-term
  product, coverage, stewardship, and participant-experience gaps.
- [Geodata scaling roadmap](../geodata/horizontal-scaling-roadmap.md) — Phases
  0–4 are complete for their documented evidence bounds. Phase 5 now has
  bounded live rollout/HPA evidence but remains open for broader capacity,
  independent-row consistency, node/storage failure, staged canary, and
  rollback qualification. See the [Phase 5 evidence](../geodata/evidence/phase5-staged-rollout-2026-10-09.md),
  [Phase 4 evidence](../geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md)
  and [roadmap](../geodata/horizontal-scaling-roadmap.md).
- [Observability coverage gap](../observability/overview.md#live-k3s-verification-and-known-gap)
  — Identity and Programme `/metrics` coverage needs remediation and live
  verification.
