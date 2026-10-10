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
  — Phases 0–4 are complete within their recorded evidence bounds. Phase 5's
  four Geodata work routes, migration 021, partial-cascade recovery and
  Activity idempotency fix are live. Helm revision 191 is deployed; Fleet is
  Ready=True at `a68eedd5ba7ee8aa0297d14ed8a38c4fceb9f109`, with 60/60 resources. Configured immutable image
  references match live pod IDs. Target durables are empty with active workers;
  four legacy Geodata durables remain empty through the rollback observation,
  ending no earlier than 21:28:41 UTC on 11 October 2026. Isolated retry, cancellation and
  expiry-to-completion checks passed; the test namespace is deleted. The only
  Phase 5 gate is the remaining observation and safe retirement of those four
  durables. Phase 6's fact-stream transition remains planned. See the
  [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
  [migration plan](../operations/messaging/nats-event-migration-plan.md),
  [work queue runbook](../operations/messaging/activity-work-queues.md), and
  [Work in progress](../work-in-progress/README.md).
- [REST API alias retirement]- [REST API alias retirement](../domain/api/rest-consolidation-plan.md) —
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
