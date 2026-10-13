# Work in progress

The latest verified runtime and repository snapshot is [Current system state](../architecture/current-state.md). This index tracks active delivery and evidence gates.

The dashboard enhancement issues are complete: the K3s workload, application
workflow, and PostgreSQL/PostGIS dashboards are deployed and verified. Their
evidence is in the [dashboard record](../observability/evidence/dashboard-enhancements-2026-10-12.md);
they are no longer active work items.

This index points to active delivery and evidence gates. Completed implementation
does not close independent rollout or acceptance work. Broader unstarted work
is listed in [To do](../to-do/README.md).

## Recently completed

- [Database migration baselines and data-preserving rebuilds](../operations/databases/baseline-migration-inventory.md) — Deploy #15/#16, Activity #4/#5, and Geodata #8/#9 are all closed. Each rebuild checklist is 7/7. The final archives passed full and baseline data-only restores, the live databases were rebuilt and verified, Helm revision 288 is deployed, and temporary restore data was removed. See the [cutover evidence](../operations/databases/database-rebuild-evidence-2026-10-13.md).

## Cross-repository architecture review

Phases 0–4 are complete and documented in [REVIEW.md](../../REVIEW.md), the
[Phase 0 closure record](../architecture/evidence/review-phase-0-2026-10-11.md),
and the [Phases 1–4 evidence record](../architecture/evidence/review-phases-1-4-2026-10-11.md).
The contract cleanup definition, duplicate-key CI gate, handler check, and
canonical/root/Platform mirrors are reconciled.

## Active implementation and evidence

- [Geodata scale qualification](../geodata/horizontal-scaling-roadmap.md) —
  Phase 5 was administratively closed by owner acceptance on 11 October. Broad
  capacity, independent writes, node/storage failure, canary, rollback, and
  scaling above three replicas remain unverified; reopen these gates before
  expanding capacity. See the [closure record](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md).
- [Operations Service retirement](../operations/evidence/operations-service-retirement-2026-10-11.md) — runtime and observability replacements are verified; repository archive remains in [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3) because this connector has no archive operation.
- [Structured logging acceptance](../observability/logging-implementation.md) —
  Helm revision 272 is deployed and Fleet is Ready with 23 Deployments and 27 replicas. Identity
  request-to-worker correlation and Identity/Programme metrics scraping are
  verified. Programme/Activity/Geodata producer-specific live correlation,
  authenticated Grafana navigation, Loki field privacy, alerts/retention/
  recovery, and Compose parity remain open in the evidence record.
