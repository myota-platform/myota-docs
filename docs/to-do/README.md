# To do

For current deployment and completed/open workstream status, see [Current system state](../architecture/current-state.md). This index lists accepted work that has not started or still needs a decision.

This index lists accepted work that has not started or still needs a decision.
The linked roadmap is the source of detailed scope and evidence. This page is
only a navigation index.

For the recommended execution order, see the
[prioritized backlog](prioritized-backlog.md).

- [Geodata scale expansion](../geodata/horizontal-scaling-roadmap.md) —
  Phase 5 is closed by owner acceptance for the current scope. Reopen its
  unverified broad-capacity, independent-write, node/storage, canary, and
  rollback gates before expanding beyond the tested replica range. See the
  [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md).
- [Structured logging acceptance](../observability/logging-implementation.md) —
  Helm/Fleet state reconciliation, authenticated Grafana navigation,
  request-to-worker correlation, Loki field privacy review, and Compose parity
  remain open in the evidence record.
- [Programme configuration gaps](../domain/programmes/configuration-gap-analysis.md) —
  ownership/lifecycle, locale/content, jurisdiction/approver, eligibility, and
  privacy/identity configuration remain roadmap work.
- [Admin UI internationalization](../domain/administration/i18n-roadmap.md)
  — proposed locale foundation and translation of administration workflows.
- [Future award-condition model](../domain/awards/future-condition-model-roadmap.md)
  — typed award conditions remain future capability work.
- [Charter delivery gaps](../governance/charter-gap-analysis.md) — longer-term
  product, coverage, stewardship, and participant-experience gaps.

- [Database data-preserving rebuilds](../operations/databases/baseline-migration-inventory.md) — baseline issues Deploy #15, Activity #4, and Geodata #8 are complete and closed. Rebuild issues #16, #5, and #9 remain open. Platform baseline mirror PR #5 is merged to `main`; Deploy #20, Activity #10, and Geodata #12 are awaiting promotion CI/merge. Do not cut over until the exact runner image is deployed and each post-quiescence archive passes isolated restore with a recorded go/no-go.
