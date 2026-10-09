# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md) —
  Phase 0 inventory, topology decision, and owner assignments are complete.
  Phase 1 is underway: contracts now list 68 domain facts and ten proposed work
  commands; a create-only, drift-checking provisioner requires explicit finite
  limits. Two contract and three topology tests pass. The per-event payload
  contracts, authenticated least-privilege NATS roles, measured capacity limits,
  restore/replay test, and removal of relay-side provisioning remain open. The
  deployed shared `MYOTA_EVENTS` stream still uses Interest retention; no
  producer/consumer path or live stream has changed. See the [Phase 1 evidence](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md)
  and [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
  Volker Kerkhoff (`@kerk1v`) is assigned, with Codex pairing support. Evidence
  work is tracked in [contracts #1](https://github.com/myota-platform/myota-contracts/issues/1),
  [Identity #1](https://github.com/myota-platform/myota-identity-service/issues/1),
  [Programme #1](https://github.com/myota-platform/myota-programme-service/issues/1),
  [Activity #1](https://github.com/myota-platform/myota-activity-service/issues/1),
  [Geodata #2](https://github.com/myota-platform/myota-geodata-service/issues/2),
  [deploy #3](https://github.com/myota-platform/myota-deploy/issues/3), and
  [Operations #1](https://github.com/myota-platform/myota-operations-service/issues/1).
  A producer or consumer path remains gated on its assigned evidence; no runtime
  change is claimed complete.
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
