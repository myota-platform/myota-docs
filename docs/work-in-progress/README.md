# Work in progress

This page lists work that is actively being implemented or has open acceptance
gates. The [current system state](../architecture/current-state.md) is the
source for live deployment status and verified evidence. Accepted work that is
not active belongs in [To do](../to-do/README.md); recommended order is in the
[prioritized backlog](../to-do/prioritized-backlog.md).

## Active work

- [Asynchronous email delivery](../operations/messaging/email-delivery.md)
  — the versioned command contract is checked in, and PR #82 adds the bounded
  email validation/rendering core plus an email-specific test/lint workflow.
  The NATS durable consumer, Identity recipient API/outbox, delivery database,
  retry/DLQ, runtime telemetry, and Compose/Helm deployment remain open. No
  email is sent by the current code.
- [Shared account profile fields and private images](../domain/identity/profile.md)
  — Contracts, Identity, Participant Web, and deployment PRs #76–#79 are
  merged to Testing. The latest check found all 24 Deployments and 3
  StatefulSets ready and all 10 PVCs bound; scoped/anonymous S3 permission
  checks passed with cleanup. Issue #12 remains open for mirror/recovery
  documentation and evidence; production has not changed.
- [Complete the monorepo release-path transition](../architecture/monorepo-and-scalability-assessment.md)
  — all eight image packages publish from monorepo Testing, and digest PR #78
  recorded the current test digests. Testing Fleet now reads `myota/testing`;
  its GitRepo currently displays `WaitApplied` while Helm revision 327 is
  `pending-upgrade`, despite ready resources. Automatic digest updates remain
  unverified because the monorepo workflow needs its scoped App credentials.
  Production still reads the former Deploy repository; archive old
  repositories only after both environment cutovers. See the
  [dated evidence](../architecture/evidence/monorepo-transition-2026-10-14.md)
  and [myota #53](https://github.com/myota-platform/myota/issues/53).
- [Structured logging and observability acceptance](../observability/logging-implementation.md)
  — remaining gates are authenticated Grafana trace/log navigation, Compose
  runtime parity, and representative capacity/recovery evidence. Correlation
  and Identity/Programme metrics work already verified as complete is recorded
  in the [current-state page](../architecture/current-state.md) and linked
  evidence. MyOTA #22 PRs #80 and #81 add finite Grafana filters; both bounded
  dashboards are mounted and Grafana logged successful provisioning. At the
  21:43 UTC check, Fleet displayed `WaitApplied` despite 76/76 resources ready;
  Helm revision 327 was still `pending-upgrade`. See the
  [test deployment record](../operations/test-deployment.md) and
  [issue #22](https://github.com/myota-platform/myota/issues/22).

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
