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
  — Phase 0 inventory, Phase 1 contracts/topology, and Phase 2 relay hardening
  are complete within their recorded evidence bounds. Phase 2 added contract-
  backed routing, read-only legacy topology validation, retry/DLQ recovery,
  backlog metrics, alerts, and isolated broker/database qualification. The
  mixed Interest stream and existing consumer/work paths remain deployed.
  Payload privacy/schema enforcement and controlled consumer/work-stream
  cutover remain open. See the [Phase 1 completion evidence](../operations/messaging/evidence/phase1-completion-2026-10-10.md),
  [Phase 2 relay evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
  [joint review](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  [recovery runbook](../operations/messaging/jetstream-recovery.md), and
  [Work in progress](../work-in-progress/README.md).
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
