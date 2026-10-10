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
  — Phases 0–3 are complete within their recorded evidence bounds. Phase 3
  deployed Activity's exact 21-subject `activity-notifications-v1` durable,
  idempotent transactional handling, audited poison-event redrive, and bounded
  metrics. The four Geodata work durables and shared file-backed Interest stream
  remain unchanged; the two broad Activity durables were retired after successor
  validation. Phase 4 Activity work migration, Phase 5 Geodata work migration,
  and the final stream-retention cutover remain open. Fleet's bundle is reporting
  `WaitApplied` after a recovery rollback even though Helm revision 170 and all
  60 resources are deployed/ready; clear this status divergence before the next
  runtime phase. See the [Phase 3 evidence](../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
  [Activity notification runbook](../operations/messaging/activity-notification-consumer.md),
  [Phase 1 completion evidence](../operations/messaging/evidence/phase1-completion-2026-10-10.md),
  [Phase 2 relay evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
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
