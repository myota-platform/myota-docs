# Geodata

The [current system state](../architecture/current-state.md) summarizes the
deployed service and its evidence limits. Geodata owns spatial entities,
imports, provenance, review, and its durable workers; see the
[repository map](../architecture/repository-map.md) for cross-service and
database ownership.

## Roadmaps and operating behavior

- [Horizontal scaling roadmap](horizontal-scaling-roadmap.md) — phase scope,
  qualification boundaries, and accepted limits. Phase 5 was closed by owner
  acceptance; unverified capacity and failure gates must be reopened before
  expanding beyond the accepted envelope.
- [Relational authority](phase1-relational-authority.md) — row ownership,
  writer fencing, and concurrency invariants.
- [Location enrichment](location-enrichment.md) — provider-backed values,
  request guards, and provenance.
- [Maidenhead locators](maidenhead-locators.md) — geometry-derived grid
  coverage.
- [Ways and line geometry](ways.md)
- [QGIS workflow](qgis-workflow.md)
- [Entity catalogue editor](../domain/administration/entity-catalogue-editor.md)
  — Website and Description fields, persistence, audit, and import precedence.

## Evidence and implementation prompts

- [Evidence index](evidence/README.md) — qualification results and explicit
  limits by phase.
- [Scaling implementation prompts](prompts/scaling/README.md) — phase-scoped
  implementation prompts and exit criteria.
