# ADR-0006: Synchronize geodata migrations from the owning service

## Status

Accepted

## Decision

`myota-geodata-service/migrations/` is the canonical source for the complete
ordered schema of the isolated `myota_geo` database. It contains the core
PostGIS schema, QGIS views, and production pipeline additions.

`myota-platform/db/migrations/geo/` is a synchronized copy for the runnable
vertical-slice bootstrap. `myota-deploy/db/migrations/geo/` is a synchronized
copy consumed by the release migration runner. Both mirrors must match the
service source byte-for-byte and must not be edited independently.

When a geodata schema change is required:

1. Change and review the ordered SQL in `myota-geodata-service/migrations/`.
2. Synchronize the complete ordered set into the platform and deployment
   mirrors.
3. Verify equality and run the migration/bootstrap tests before committing.

Shared tables for service state, idempotency, outbox delivery and consumer
bookkeeping belong to the core migration. They are not repeated in the
geodata migration set.

## Rationale

The service remains the owner of its data model and schema evolution, while
the platform bootstrap and deployment repository can run a complete database
installation without importing service repositories at runtime. A single
service-owned source avoids divergent SQL; explicit mirrors preserve the
operational convenience of centralized migration ordering.

## Consequences

- Geodata schema review stays with the geodata service.
- Platform and deployment changes must include synchronized migration copies.
- Release automation can apply core and geodata databases in a deterministic
  order.
- Cross-database references remain opaque IDs and events rather than foreign
  keys.
