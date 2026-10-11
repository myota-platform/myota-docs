# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md)
  — **Status:** Phases 0–6 are complete within their recorded evidence bounds.
  Helm revision 213 is deployed; the MyOTA Fleet bundle is Ready=True on Deploy
  main commit `dfc423d26bd79a82b1177b9e5d51e2d1196580c9`, and every MyOTA
  Deployment has its desired replicas ready. `MYOTA_EVENTS` is a bounded,
  file-backed Limits fact stream with only `myota.events.>`; Activity and
  Geodata work remain on their separate WorkQueue streams. The one-time
  migration flag is disabled and regular topology drift validation remains on.
  - **Phase 6 verification:** Deploy Python quality and Helm validation passed
    (runs 38099669717 and 38099669405). The disposable host-K3s migration
    and idempotent-rerun check passed without publishing a message; its
    namespace was removed. All live stream, durable, relay/outbox, migration
    flag, and Deployment checks passed.
  - **Preserved limits:** Six existing Geodata database DLQ records remain
    unchanged. The 24-hour Phase 5 observation was explicitly waived and
    closed early, not represented as a completed 24-hour window. Previously
    Interest-expired facts cannot be recovered from JetStream; PostgreSQL
    remains authoritative and off-node recovery is deferred. No production
    test data, metrics baseline, shadow traffic, or canary was used. See the
    [Phase 6 evidence](../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md),
    [Phase 5 evidence](../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
    and [live diagram](../architecture/diagrams/nats-event-migration.md).
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
  — deployed storage-status and Grafana access evidence; records the test
  scope and remaining logging gap. The proposed NATS Surveyor consolidation is
  tracked separately in its observability roadmap.
- [Recent award designer delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md)
  — UI/API verification record, including the platform integration test not run
  against an isolated NATS broker.
- [Recent entity catalogue delivery evidence](../domain/administration/evidence/entity-catalogue-2026-10-09.md)
  — browser, build, and deployment evidence for the catalogue workflow.
