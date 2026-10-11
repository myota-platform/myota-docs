# To do

This index lists accepted backlog and proposed work that has not started, or
that still needs a decision. Check the linked roadmap for item-level status and
evidence; this page is a navigation index, not a second source of truth.

For a recommended execution order and the reasons behind it, see the
[prioritized backlog](prioritized-backlog.md). Priority is based on the current
documentation evidence and does not set delivery dates.

- [Cross-repository architecture remediation](../../REVIEW.md) — begin with the
  duplicate destructive OpenAPI definition and owner-to-Deploy/Platform source
  parity; later phases cover shared helpers, Geodata complexity, and measured
  REST alias retirement.
- [Future award-condition model](../domain/awards/future-condition-model-roadmap.md)
  — define and implement typed POTA-like, worked-entity, Maidenhead, geographic,
  and diversity awards.
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
- [Structured logging acceptance](../observability/logging-implementation.md) —
  deployed with active Operations PostgreSQL metrics and SQL-scrubbed Psycopg
  traces. Stable NATS work/event correlation is implemented. Remaining
  acceptance is Helm/Fleet release-state reconciliation, authenticated Grafana
  navigation, request-to-worker correlation, Loki field privacy audit, and
  Compose runtime parity. See
  [evidence](../observability/evidence/logging-implementation-2026-10-11.md).
- [Observability coverage gap](../observability/overview.md#live-k3s-verification-and-known-gap)
  — Identity and Programme `/metrics` coverage needs remediation and live
  verification.
