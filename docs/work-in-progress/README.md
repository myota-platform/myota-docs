# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md) —
  Phase 0 inventory and topology decisions are complete. Phase 1 contract work
  now has 68 registered facts and ten selected work commands, checked-in
  schemas, source audit, and a passing contracts CI check that each work command
  matches its deploy-owned stream, subject, and durable. The create-only
  provisioner and drift checks pass focused tests and an isolated broker
  idempotency/drift check.
  The 10 October delegated review dispositioned all 68 source-derived schemas
  and selected the cluster-internal trust boundary, capacity, recovery, and relay ownership. That
  decision review is complete; runtime gates remain for payload minimization,
  representative capacity sizing, off-node restore and database reconciliation,
  payload minimization, unknown-route enforcement, and a safe provisioning/readiness
  barrier before relay mutation is removed. NATS auth/TLS is not required while
  the broker remains cluster-internal. The
  [recovery runbook](../operations/messaging/jetstream-recovery.md) records the
  procedure and remaining qualification evidence. The live shared stream still
  uses Interest retention and no producer or consumer path has changed. See the
  [joint review](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  [Phase 1 evidence](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md),
  and [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
  Phase 1 remains open until all exit criteria in the
  [migration plan](../operations/messaging/nats-event-migration-plan.md) pass.
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
