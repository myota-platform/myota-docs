# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md)
  — **Status:** Phases 0–3 are complete within their evidence bounds. Phase 4
  source implementation and isolated qualification are complete; production
  cutover remains active.
  - **Phase 4 implementation:** Six Activity commands use ID-only work
    envelopes, transactional job/outbox writes, exact per-kind pull durables,
    post-commit ACK, retry/backoff, renewable token-fenced leases, persisted
    work dead letters, and audited database redrive. The first migration run
    rejects old poll-claimed jobs still `RUNNING`; it backfills queued work and
    drops the obsolete poller index while preserving `activity_job` status and
    history. `NOTIFICATION_SEND` rows are corrected to delivered and its
    synthetic jobs are removed.
  - **Isolated evidence:** 37 Activity tests passed (one optional broker test
    skipped); contract tests passed 7/7; the source audit covered 68 facts and
    16 legacy work types; deploy tests passed 25/25; Helm lint passed. A
    disposable host-K3s PostgreSQL/NATS namespace verified migration guard and
    rerun, lease fencing, terminal DLQ/redrive, all six durables, ACK state,
    and an empty WorkQueue after processing. The namespace and port forwards
    were removed.
  - **Production baseline:** Read-only checks found 154 synthetic
    `NOTIFICATION_SEND` jobs, all succeeded; no selected Activity work jobs or
    queued notifications; the old claim index remains; `MYOTA_ACTIVITY_WORK`
    and `MYOTA_GEODATA_WORK` are absent; mixed `MYOTA_EVENTS` is unchanged.
    Helm 173 is deployed and Fleet reports myota-deploy Ready at `81c2f321`.
  - **Remaining gates:** staged stream provisioning; Activity worker scale to
    zero and pod drain; guarded production migration; new image rollout and
    per-kind durable/metric verification; then disable the transitional
    missing-outbox repair after all old API pods are gone. Preserve Geodata
    queues and mixed-stream retention during this phase.
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
