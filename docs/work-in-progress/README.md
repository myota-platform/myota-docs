# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md)
  — **Status:** Phase 0 and Phase 1 contract/topology work are complete.
  - **Phase 1 evidence:** 68 facts and ten work commands are registered with
    bounded per-command payload schemas. Source and contract checks passed.
    The optional Helm pre-upgrade readiness hook fails closed.
  - **Local qualification:** Isolated K3s checks covered provisioner
    idempotency and drift, local PVC restore, replay, and capacity rejection.
    The 30-day database sample is short; accepted initial caps and that evidence
    limitation are recorded. Off-node recovery is deferred at the project
    owner's direction.
  - **Phase 2 gates:** Runtime enforcement, payload minimization, relay mutation
    removal, and production cutover remain open.
  - **Current boundary:** NATS remains cluster-internal without auth/TLS. The
    live shared stream still uses Interest retention; no producer or consumer
    path has changed.
  - **References:** [Phase 1 completion evidence](../operations/messaging/evidence/phase1-completion-2026-10-10.md),
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
