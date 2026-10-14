# Diagram index

These Mermaid diagrams explain current runtime ownership and selected architecture.
The service boundaries and deployment are unchanged, but source ownership is
transitioning separately. The service-boundary diagram distinguishes candidate
monorepo source from the split-repository release that is still deployed.
Diagrams do not prove that every pictured capability is deployed. The
[current-state snapshot](../current-state.md) summarizes the latest live checks;
linked plans and evidence remain authoritative for their scope. Sections labeled
historical or proposed are snapshots or decisions, not current runtime claims.

## Current platform snapshot

- [Current system state](../current-state.md)

## Platform and domain data

- [Service boundaries and databases](service-boundaries.md)
- [Domain data model](data-model.md)
- [Programme configuration lifecycle](programme-configuration-lifecycle.md)
- [Geodata category assignment](geodata-category-assignment.md)
- [Catalogue editor navigation and persistence](entity-catalogue-editor.md)
- [Administration workspace navigation and permission boundaries](admin-workspaces.md)

## Geodata lifecycle

- [Geodata candidate/import/review lifecycle](geodata-resource-lifecycle.md)
- [Import validation and promotion queues](geodata-import-validation.md)
- [Object-storage buckets and retention](object-storage-buckets.md)

## Activity and operations

- [Activity and award jobs](activity-award-jobs.md)
- [API Phase 4 client and operational migration](api-phase4-operational-migration.md)
- [OpenTelemetry observability architecture](observability-architecture.md)
- [NATS current and selected event/work topologies](nats-event-migration.md) —
  live deployment versus the ADR-0008 target, with phase status.
