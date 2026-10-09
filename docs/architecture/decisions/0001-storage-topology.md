# ADR-0001: Three service-owned databases

## Decision

Use three service-owned databases: `myota_core` for identity, programmes,
configuration and shared control-plane data; `myota_activity` for
activations, QSOs, aggregates, awards and execution jobs; and `myota_geo` for
geometry, imports, provenance and review. Run core and activity on plain
PostgreSQL and install PostGIS only for geodata. Local Compose uses three
database containers. Kubernetes receives three independently addressable
database targets through the database secret.

## Why

Geodata has different workload characteristics: large bulk imports, spatial
indexes, geometry-heavy backups, conflation jobs and map reads. Activity has
high-write QSO ingestion and independent worker concurrency. Core contains
lower-volume control-plane transactions. Separate databases provide workload,
backup, migration and scaling isolation while keeping each service's schema
and connection pool explicit.

## Alternatives considered

- One database/schema: simplest operations, but bulk imports and spatial maintenance can compete with logins/activations; backup/restore cannot be isolated.
- Two databases (`core` + `geo`): better spatial isolation, but activity QSO writes still compete with control-plane transactions and activity migrations remain misleadingly coupled to core.
- Separate cluster from day one: strongest failure and scaling isolation, but triples baseline operations for a small deployment.
- Non-PostGIS geodata store: rejected because geometry validation, spatial indexes, conflation and graphical editing are first-class requirements.

## Consequences

Cross-database relations are opaque IDs and events, not foreign keys. A later
move from containers to managed instances or separate clusters is
operationally straightforward but requires event/retry monitoring. QGIS and
browser admin tools connect to `myota_geo` with a least-privilege editing
role; public APIs expose reviewed records only. The local migration runner
copies existing activity tables from the old core database and geodata tables
from the old core-hosted `myota_geo` database into empty targets, and is
idempotent. The legacy geodata database is retained until counts are verified.
