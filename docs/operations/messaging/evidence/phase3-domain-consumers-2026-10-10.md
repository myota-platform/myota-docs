# Phase 3 domain-consumer evidence — 10 October 2026

## Scope and decision

Phase 3 covers consumers of committed domain facts. The contracts registry
selects one group: Activity notifications for 19 Identity facts and two
Geodata review/status facts. No Programme subscriber was added. Activity writes
only its local notification projection. Producer routes, Geodata work
consumers, Activity database-polled jobs, and Operations' read-only broker
inspection boundary remain unchanged.

The group uses the stable pull durable `activity-notifications-v1` with the
exact 21 registered subjects. It uses explicit ACK, `DeliverPolicy.ALL`,
`ReplayPolicy.INSTANT`, a 60-second ACK wait, eight deliveries, 64 maximum
ack-pending messages, 32 maximum waiting pulls, and bounded backoff. Handler
effects, `consumer_processed_event`, and the consumer checkpoint commit in the
same Activity database transaction before ACK. The notice projection stores
only event type and event ID. The unique `event:<eventId>` key makes redelivery
idempotent.

Poison messages are stored in Activity's database with stream/sequence
coordinates and a redacted payload. The bounded metrics contain outcome labels
only; unresolved dead-letter count is a gauge. Operators can redrive a message
only through the audited command, which records actor and reason and preserves
the original event ID. See the
[Activity notification runbook](../activity-notification-consumer.md).

## Source and contract changes

- `myota-activity-service` commit
  [`52f7cfc`](https://github.com/myota-platform/myota-activity-service/commit/52f7cfcf3fefc2ccfc82d660f7b42af6cf1faec9)
  adds the transactional subscriber, topology helper, audited redrive, schema
  migration, metrics, and focused tests.
- `myota-deploy` commit
  [`6b8cded`](https://github.com/myota-platform/myota-deploy/commit/6b8cdeddd41766e564856580c24075cfd7fbd400)
  adds Helm provision/retire hooks, metrics wiring, alerting, and dashboard
  updates. Follow-up commit
  [`4f2aa45`](https://github.com/myota-platform/myota-deploy/commit/4f2aa45eb5b735abef28b59ca3ff414cee0b9e5a)
  preserves validation compatibility for the unchanged Geodata durables.
- `myota-deploy` digest commit
  [`81c2f32`](https://github.com/myota-platform/myota-deploy/commit/81c2f3217b67a91a104c1b4797cfd9fa4c0e83ca)
  pins the published Activity and migration image digests. The Activity image
  was built from the Phase 3 source; the platform image includes the Activity
  database migration.
- `myota-platform` mirror commit
  [`1bd2504`](https://github.com/myota-platform/myota-platform/commit/1bd25043cfe8571652da5dae6dff03d64d331f6b)
  synchronizes the Phase 3 implementation; commit
  [`5b7a8b2`](https://github.com/myota-platform/myota-platform/commit/5b7a8b21321eecd8f83664fa2074cd651eb9409c)
  synchronizes the relay compatibility fix.
- `myota-contracts` commit
  [`c125a26`](https://github.com/myota-platform/myota-contracts/commit/c125a26c904937367b8624b295389905a1ea713f)
  asserts that service and relay filters match the registry. Formatting fix
  [`c29cebd`](https://github.com/myota-platform/myota-contracts/commit/c29cebd5888f2ad3bbe583ef8e35963c8e395df8)
  passed Contracts CI.

## Verification

- Activity source checks passed: Ruff on changed files; 29 unit tests passed,
  with one broker test skipped when `NATS_TEST_URL` was absent. The separate
  disposable K3s/PostgreSQL/NATS integration verified the exact filter set,
  commit-success/ACK-loss idempotency, redacted poison capture and audited
  replay, zero unresolved-DLQ metric after processing, and unsubscribe/drain
  shutdown.
- Deploy checks passed: Ruff, five focused relay retention tests, Helm
  rendering, and YAML/JSON parsing for the collector, alert rules, and
  dashboard. GitHub Actions passed Python quality and Helm validation on
  [`81c2f32`](https://github.com/myota-platform/myota-deploy/actions/runs/38066035827)
  and [`81c2f32` Helm validation](https://github.com/myota-platform/myota-deploy/actions/runs/38066035190).
- Contracts registry and workspace-audit checks passed in the contract workflow
  on [`c29cebd`](https://github.com/myota-platform/myota-contracts/actions/runs/38066650821).
  The initial contract commit's workflow had a Ruff-only formatting failure;
  the formatting correction was pushed and the rerun passed.
- The isolated test namespaces were deleted. No test event or domain row was
  inserted into the production databases or broker.

## Live K3s verification

The local K3s cluster reports Helm release `myota` deployed at revision 170 on
chart `myota-0.2.14`. All 60 resources are reported ready; the Activity
notification Deployment and each of the three relay Deployments are ready at
1/1. A live JetStream API metadata query observed:

- `MYOTA_EVENTS` is still file-backed with Interest retention and subjects
  `myota.events.>` and `myota.geodata.>`.
- The stream has five consumers: `activity-notifications-v1` with the exact 21
  registered subjects, plus the four unchanged Geodata work durables.
- Activity notifications report zero pending, zero ack-pending, and zero
  redelivered messages; one pull request is waiting. The two broad Activity
  durables are absent. No stream retention, message, or work durable was
  changed by Phase 3.
- The Activity unresolved-notification-dead-letter gauge is zero.
- Normal chart migration Job `myota-migrations-169` completed successfully with
  the new migration image. The first rollout's migration Job had used the
  previous digest; to unblock that in-progress rollout, the exact additive,
  idempotent Activity migration was applied once by a short-lived Job in
  `myota`, then the normal revision-169 migration hook reapplied it. The repair
  Job was deleted after verification. No database or JetStream test data was
  created.

The Fleet GitRepo has fetched deploy commit `81c2f32` and reports 60/60 ready
resources, but its bundle condition is still `WaitApplied` following recovery
from the failed first attempt. The Helm release is deployed and the live
consumer/configuration checks pass; clear this Fleet status divergence before
starting another runtime phase.

## Exit criteria and remaining gates

Both Phase 3 exit criteria are met: the only selected domain-event group is
live, documented, independently deployable, idempotent, observable, and tested
against redelivery/restart; and no unsupported broad Activity consumer remains
to determine stream retention accidentally. This completes Phase 3 within its
consumer scope.

Phase 4 remains gated on Fleet reporting a reconciled bundle. Later work must
still close payload privacy/schema enforcement and Geodata payload
minimization, migrate the selected Activity and Geodata work commands, qualify
capacity and replay, and plan the mixed Interest-to-Limits transition. The
shared stream remains a bounded delivery mechanism, not an event archive.
