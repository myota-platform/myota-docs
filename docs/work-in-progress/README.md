# Work in progress

This index points to active delivery and verification threads. Completed
components may appear here when their evidence still defines an open rollout or
qualification gate. Broader unstarted items are listed in [To do](../to-do/README.md).

- [NATS migration evidence and implementation](../operations/messaging/nats-event-migration-plan.md) —
  Phase 0 inventory, topology decision, and owner assignments are complete.
  Phase 1 is underway: contracts now list 68 domain facts and ten proposed work
  commands; a create-only, drift-checking provisioner requires explicit finite
  limits and validates delivery, replay, waiting-pull, and consumer state settings.
  Four focused tests and Ruff format/lint pass in deploy and its platform mirror;
  isolated JetStream provisioning and its idempotent rerun pass. The registry maps
  all 68 fact types to source files, and its workspace audit found no
  undispositioned Python event-like source literals across five service
  repositories; the contracts main-branch source-audit workflow passed. Latest deploy and platform image checks pass. The delegated review dispositioned
  all 68 schemas and selected credential, capacity, recovery, and provisioning
  policies. Runtime gates remain: Geodata payload minimization, credential/TLS
  implementation, representative capacity evidence, restore/replay, chart
  provisioner readiness, and relay mutation removal. The deployed shared stream
  retains Interest policy; no producer/consumer path changed. See the
  [joint review](../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
  [Phase 1 evidence](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md),
  and [current/target diagrams](../architecture/diagrams/nats-event-migration.md).
  Payload schemas for all 19 Identity, 12 Programme, 10 Activity, and 27 Geodata facts are now
  derived from producer callsites and classified for sensitive/internal content.
  Identity, Programme, Activity, and Geodata schemas pass Contracts CI; the
  platform mirror CI also passes. Joint owner/privacy review remains open. Geodata
  preprocessing currently includes internal `_records` and `_status` in the
  preprocessed event payload and needs minimization review. Operations has no event-producing call sites in the source audit; an Operations
  payload schema is not applicable unless it becomes a producer. Authenticated credentials, capacity limits, recovery qualification, and relay-side
  topology mutation remain open. The contracts, deploy, platform mirror, and organization profile PRs (#2, #4, #1,
  and #1) are merged. Phase 1 remains in progress: owner/privacy review, Geodata payload minimization, authenticated
  least-privilege roles, capacity limits, recovery qualification, and relay-side
  topology mutation are still open.
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
