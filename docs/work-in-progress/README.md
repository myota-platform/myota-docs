# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md)
  — **Status:** Phases 0–5 are complete within their recorded evidence
  bounds. Phase 5's Geodata work migration, migration 021, retry-safe recovery,
  Activity idempotency fix, and immutable-image rollout are live. Helm revision
  193 remains deployed; Fleet is Ready=True at Deploy commit
  cfecd655d9c0eee9d19db26725fb11c99366815a. The four legacy Geodata
  durables were retired after final filter, backlog, migration, recovery-age
  and Fleet checks. The user waived the 24-hour elapsed-time requirement;
  closure occurred about ten minutes after the revision 193 readiness anchor,
  and is not represented as a full 24-hour observation. The Activity
  notification durable, shared Interest-retained MYOTA_EVENTS, four target
  Geodata durables, migration markers and authoritative recovery data remain.
  Phase 6 fact-stream retention is next.
  - **Phase 5 verification:** Geodata (154 tests, one optional skip), Activity
    (40 tests, one optional broker skip), and relay/topology (25 tests) passed.
    Database refusal/reconnect, same-node NATS PVC restart, two-database
    cascade retry, concurrent idempotency, cancellation race, connected
    expiry/recompletion and isolated exact-consumer deletion semantics were
    exercised. Two full delivery-suite attempts each had a timing-sensitive
    immediate ACK-counter assertion; focused ACK verification passed, but the
    full suite is not claimed as a clean pass.
  - **Cleanup:** Both disposable Phase 5 namespaces were deleted and verified
    absent. No production test data or broker messages were created.
- [NATS Surveyor monitoring consolidation](../observability/nats-surveyor-migration.md)
  — **Status: planned; implementation not started.** The current Admin UI page,
  Operations broker sampler and seven-day snapshot table, Geodata broker-metric
  poller, and MyOTA Grafana NATS dashboard remain in place. The roadmap calls
  for a pinned Surveyor deployment and dashboard compatibility proof first,
  followed by a seven-day overlap and controlled retirement of duplicate
  broker inspection. Phase 5 event/work migration is complete under its
  documented early-close waiver; Phase 6 fact-stream retention is a separate
  track. No runtime components have been removed for this item.
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
