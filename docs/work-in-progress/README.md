# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md)
  — **Status:** Phases 0–4 are complete within their recorded evidence
  bounds. Phase 5 Geodata cutover and migration 021 are live; the new image
  keeps partial Activity/Geodata deletion retryable. The four old Geodata
  durables are empty and retained through the 24-hour rollback observation.
  The Fleet check and two-database failure/replay chains are verified. The
  committed Activity idempotency fix still needs image deployment; the rollback
  observation and old durable retirement remain open. Phase 6 fact-stream
  retention remains planned.
  - **Phase 5 production:** `MYOTA_GEODATA_WORK` is a finite file-backed
    WorkQueue with four exact pull durables. Migration 021's recovery
    columns/indexes are present, and the live un-dispatched work count is zero.
    Helm revision 189 is deployed; Fleet reports Ready=True with 60/60
    resources. Geodata, shared runtime, and Activity images remain digest-pinned.
    The Geodata suite passed 154 tests (one optional test skipped); Activity's
    40-test suite passed (one optional broker test skipped); 25 relay/topology
    tests passed. Real database refusal/reconnect and same-node NATS PVC restart
    passed.
  - **Phase 5 retry qualification:** A disposable host-K3s test used both real
    service databases, the Activity API, the Geodata handler, and a private
    JetStream stream. Injected Geodata failure NAKed the delivery; redelivery
    completed with exactly one Activity cascade outbox fact and zero pending
    messages. Activity's Idempotency-Key fix is committed and mirrored, but its
    image build/publish and production rollout remain open.
  - **Phase 5 remaining gates:** Keep the four legacy durables through
    20:58:36 UTC on 11 October 2026, then recheck topology/recovery and retire
    only those four. The connected cancellation and expiry/recompletion checks
    passed. Activity image deployment and final production recheck remain open.
    The isolated namespace is deleted. Preserve migration 021 and authoritative
    job/outbox/history rows.
  - **References:** [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
    [Phase 5 plan](../operations/messaging/nats-event-migration-plan.md), and
    [migration diagram](../architecture/diagrams/nats-event-migration.md).
  - **Phase 4 implementation:** Six Activity commands use ID-only work
    envelopes, transactional job/outbox writes, exact per-kind pull durables,
    post-commit ACK, retry/backoff, renewable token-fenced leases, persisted
    work dead letters, and audited database redrive. The first migration run
    rejects old poll-claimed jobs still `RUNNING`; it backfills queued work and
    drops the obsolete poller index while preserving `activity_job` status and
    history. `NOTIFICATION_SEND` rows are corrected to delivered and its
    synthetic jobs are removed.
  - **Isolated evidence:** 38 Activity tests passed (one optional broker test
    skipped); contract tests passed 7/7; the source audit covered 68 facts and
    16 legacy work types; deploy tests passed 25/25; Helm lint passed. A
    disposable host-K3s PostgreSQL/NATS namespace verified migration guard and
    rerun, lease fencing, terminal DLQ/redrive, all six durables, ACK state,
    and an empty WorkQueue after processing. The namespace and port forwards
    were removed.
  - **Production verification:** deploy commit `480c2031589e4947ecef6ba8940b54e60af77ed1`
    provisioned `MYOTA_ACTIVITY_WORK`; the six durables report zero pending,
    ack-pending, and redeliveries with active pull waiters. Migration 007 removed
    `activity_job_claim_idx`, created the status/kind index and lease/DLQ/audit
    schema, delivered 154 in-app notification rows, and purged their synthetic
    jobs. Two workers and three APIs run the pinned Activity image; the
    compatibility repair is disabled. Prometheus returns per-kind zero series.
    No production Activity work was queued or executed during cutover.
  - **Remaining gates:** Phase 5 Geodata work migration, Phase 6 fact-stream
    retention transition, broader payload privacy/schema enforcement, and the
    accepted off-node recovery deferral. The mixed fact stream and Geodata
    work routes were unchanged in Phase 4.
  - **References:** [Phase 4 evidence](../operations/messaging/evidence/phase4-activity-work-2026-10-10.md),
    [Activity work queue runbook](../operations/messaging/activity-work-queues.md),
    [Phase 3 evidence](../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md),
    [Phase 1 completion evidence](../operations/messaging/evidence/phase1-completion-2026-10-10.md),
    [Phase 2 relay evidence](../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md),
    [recovery runbook](../operations/messaging/jetstream-recovery.md), and
    [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
- [Geodata scale qualification](../geodata/horizontal-scaling-roadmap.md) —
  Phase 2 upload/API/SeaweedFS restart recovery is verified for the recorded
  image digest. Phase 3 bounded parser/RSS, snapshot, and worker recovery gates
  are closed with a 33-test isolated evidence run and documented in the
  [Phase 3 report](../geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md).
  Phase 4 bounded API replica-safety is verified. Phase 5 now has live proof of
  API replacement during a 22.5 MB accepted import and CPU HPA scale-up/down,
  plus modest two-vs-three replica latency evidence; broader capacity,
  independent-row contention, node/storage failure, canary and rollback gates
  remain open. See the [Phase 5 report](../geodata/evidence/phase5-staged-rollout-2026-10-09.md),
  [Phase 4 report](../geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md),
  the [Phase 2 evidence](../geodata/evidence/phase2-upload-recovery-2026-10-09.md)
  and [Phase 3 implementation/exit review](../geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md).
- [Geodata Phase 0 evidence set](../geodata/evidence/phase0-production-evidence-2026-10-08.md)
  — current measured 2,875-entity baseline, with explicit limits on broader
  capacity claims.
- [Observability live status](../observability/overview.md#live-k3s-verification-and-known-gap)
  — track service metrics remediation and current K3s verification.
- [Observability and storage verification](../observability/evidence/2026-10-09.md)
  — deployed storage-status and Grafana access evidence; records test scope and
  the unqualified logging/NATS areas.
- [Recent award designer delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md)
  — UI/API verification record, including the platform integration test not run
  against an isolated NATS broker.
- [Recent entity catalogue delivery evidence](../domain/administration/evidence/entity-catalogue-2026-10-09.md)
  — browser, build, and deployment evidence for the catalogue workflow.
