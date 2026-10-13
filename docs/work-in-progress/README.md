# Work in progress

This page lists work that is actively being implemented or has open acceptance
gates. The [current system state](../architecture/current-state.md) is the
source for live deployment status and verified evidence. Accepted work that is
not active belongs in [To do](../to-do/README.md); recommended order is in the
[prioritized backlog](../to-do/prioritized-backlog.md).

## Active work

- [Activate testing-image digest updates for the isolated test release](../operations/test-deployment.md)
  — the release is deployed and ready, but GitHub's scheduled digest workflow
  is only present on Deploy `testing`; scheduled workflows run from default
  branch `main`. Resolve this through a safe release PR without promoting the
  unrelated divergent deployment changes.
- [Structured logging and observability acceptance](../observability/logging-implementation.md)
  — remaining gates are authenticated Grafana trace/log navigation, Compose
  runtime parity, and representative capacity/recovery evidence. Correlation
  and Identity/Programme metrics work already verified as complete is recorded
  in the [current-state page](../architecture/current-state.md) and linked
  evidence.

## Reopen only before expanding Geodata capacity

Geodata Phase 5 was closed by owner acceptance. Broad capacity, independent
writes, node/storage failure, canary, rollback, and scaling above three replicas
remain unverified. Reopen the relevant qualification gates before expanding
beyond the accepted envelope. See the [closure decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md).

## Recently completed

- [Operations Service retirement and repository archive](../operations/evidence/operations-service-retirement-2026-10-11.md)
  — runtime functions are retired, the GitHub repository was archived on
  2026-10-13, and [Operations issue #3](https://github.com/myota-platform/myota-operations-service/issues/3) is closed.

- [Database baselines and data-preserving rebuilds](../operations/databases/baseline-migration-inventory.md)
  — Deploy #15/#16, Activity #4/#5, and Geodata #8/#9 are closed. Each rebuild
  checklist is 7/7; final archive, restore, live cutover, and cleanup evidence
  is in the [rebuild record](../operations/databases/database-rebuild-evidence-2026-10-13.md).
- [Cross-repository architecture review](../../REVIEW.md) — Phases 0–4 are
  complete, including duplicate-YAML-key rejection and the final handler/mirror
  check. See the [phase evidence](../architecture/evidence/review-phases-1-4-2026-10-11.md).
- [NATS migration and monitoring consolidation](../operations/messaging/nats-event-migration-plan.md)
  — Phases 0–6 and Surveyor deployment are complete within their documented
  limits. See the [current-state page](../architecture/current-state.md) for
  accepted evidence gaps.
