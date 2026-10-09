# Documentation index

This is the detailed navigation page for the MyOTA documentation set. The
repository-root [README](../README.md) is the short project overview; this page
groups the source documents by purpose and by implementation lifecycle.

Status in plans and evidence:

- `[x]` — implemented or evidence-backed as stated.
- `[ ]` — not implemented, not verified, or an explicit exit criterion still open.
- A completed implementation checkbox does not imply production capacity or
  launch readiness; read the linked evidence and limitations.

## 1. Mission, governance, and current gaps

- [Reconstructed cross-repository implementation timeline](changes.md)
- [Project purpose, motivation, and charter](project-charter.md)
- [Charter gap analysis and delivery sequence](charter-gap-analysis.md)
- [Programme configuration gap analysis](programme-configuration-gap-analysis.md)
- [Documentation status reconciliation](documentation-reconciliation-2026-10-07.md)
- [Original source inspection](source-inspection.md)
- [Migration from MPOTA](migration-from-mpota.md)
- [Organization profile and roadmap](https://github.com/myota-platform/.github/tree/main/profile)

## 2. System architecture and ownership

- [Overall system architecture](architecture.md)
- [Repository map, service ownership, and deployment boundaries](repository-map.md)
- [Three-database topology and migration](adr/0007-three-database-migration.md)
- [Service/repository shape](adr/0003-service-and-repository-shape.md)
- [Storage topology](adr/0001-storage-topology.md)
- [Architecture decision index](adr/README.md)
- [Architecture diagrams](diagrams/README.md)
- [NATS JetStream event and work-queue migration plan](nats-event-migration-plan.md)

## 3. Domain, API, and client contracts

- [REST API consolidation plan and phased status](api-rest-consolidation-plan.md)
- [API contract freeze](api-contract-freeze.md)
- [Phase 1: low-risk resource updates](api-phase1-resource-updates.md)
- [Phase 2: geodata resource model](api-phase2-geodata-resource-model.md)
- [Phase 3: activity and award jobs](api-phase3-activity-award-jobs.md)
- [Phase 4: client and operational migration](api-phase4-client-operational-migration.md)
- [Entity category catalogue](entity-categories.md)
- [Programme policy ownership](adr/0004-programme-policy-ownership.md)
- [UTC time policy across clients, APIs and observability](utc-time-policy.md)
- [Programme configuration gaps](programme-configuration-gap-analysis.md)
- [Activity, awards, and programme execution](awards-and-programme-execution.md)
- [Future award-condition model and implementation roadmap](award-condition-model-and-roadmap.md)
- [Programme editor, artwork uploads and certificate previews](programme-and-award-design.md)
- [Identity, callsigns, and security](identity-security.md)
- [Administration web UX](admin-web-ux.md)
- [Entity catalogue editor and geometry workspace](entity-catalogue-editor.md)

## 4. Geodata lifecycle and scaling

- [Geodata horizontal-scaling roadmap](geodata-horizontal-scaling-roadmap.md)
- [Phase 0: production baseline and evidence record](geodata-phase0-production-evidence-2026-10-08.md)
- [Phase 0: representative PostGIS, API, storage, and worker review](geodata-phase0-representative-query-review-2026-10-08.md)
- [Pre-fixture low-cardinality PostGIS plan snapshot](geodata-phase0-current-catalogue-plan-2026-10-08.md)
- [Large-upload / SeaweedFS / worker correlation](geodata-phase0-storage-correlation-2026-10-08.md)
- [Load-test and query-plan runbook](geodata-load-test-and-query-evidence.md)
- [Resumable-upload verification](geodata-load-test-upload-verification.md)
- [Phase 1 relational authority and concurrency](geodata-phase1-relational-authority.md)
- [Location enrichment](geodata-location-enrichment.md)
- [Maidenhead locator fields and geometry coverage](geodata-maidenhead-locators.md)
- [Line and way geometry support](geodata-ways.md)
- [QGIS editing workflow](qgis-workflow.md)
- [Scaling implementation prompt index](geodata-scaling-prompts/index.md)

The Phase 0 exit criterion is deliberately separate from implementation
checklists: it is complete for the measured 2,875-entity catalogue after
representative map/catalogue plans and bounded read evidence were reviewed
together. This does not qualify a larger user, entity, or QSO scale. The current
permanent fixture set contains 10,000 synthetic input records in four imports
of 2,500. The intended promotion sample is 5% from each import (2.5% Candidate,
2.5% Approved; 250 entities per status over a fresh four-import run). One
first import was already queued for full approval before this split was
requested; the remaining imports use the revised sample, and the exception is
tracked in the linked evidence record.

## 5. Operations, storage, and observability

- [Operations and recovery runbooks](operations.md)
- [OpenTelemetry architecture and service metrics](observability.md)
- [Logging implementation roadmap](observability/README.md)
- [JetStream admin status page and durable history](jetstream-admin-status.md)
- [SeaweedFS admin storage status, sampled history and Grafana Editor access](seaweedfs-admin-status.md)
- [Object-store bucket and retention boundaries](diagrams/object-storage-buckets.md)
- [Storage migration/synchronization decision](adr/0007-seaweedfs-object-storage.md)
- [Python code quality and local hooks](development/README.md)
- [Production core and environment setup](production-core.md)
- [GitHub organization handoff](github-handoff.md)

Current live observability gap: Identity and Programme pods are Ready, but
their configured `/metrics` endpoints returned 404 during the 8 October K3s
verification; their service/API metrics are not present in Prometheus. The
geodata Phase 0 evidence is unaffected. See the
[observability status and remediation note](observability.md#live-k3s-verification-and-known-gap).

## 6. Security

- [Security documentation index](security/README.md)
- [Threat model](security/threat-model.md)
- [Identity and account security](identity-security.md)
- [Role and geodata administration controls](adr/0007-administration-role-and-geodata-controls.md)

## 7. Plans and implementation status

Use the roadmap/checklist in the document that owns a feature. Do not treat an
architecture description as a completion claim. Current cross-cutting status:

- Production-core, identity, administration, geodata import/review, activity,
  API migration, and observability implementations have their own detailed
  delivery records and explicit remaining work.
- Geodata scaling Phase 0 has a completed evidence gate at the measured
  2,875-entity fixture size. It does not establish broader capacity; later
  scaling phases and larger user/QSO workloads remain open.
- Programme-configuration gaps remain an audit/to-do list, not a statement that
  those features are implemented.
- REST legacy-alias retirement remains open until usage is measured and clients
  have migrated.

See [the organization profile](https://github.com/myota-platform/.github/tree/main/profile)
for the consolidated organizational roadmap.
