# ADR-0007: Three database targets and first-run data migration

## Status

Accepted

## Decision

The platform has three database targets with explicit ownership:

| Database | Engine | Owner | Contents |
|---|---|---|---|
| `myota_core` | PostgreSQL | identity/programme services | accounts, callsigns, roles, programmes, policies, shared control-plane state and core outbox |
| `myota_activity` | PostgreSQL | activity service | activations, QSOs, aggregates, awards, imports, jobs, statistics, notifications and activity outbox |
| `myota_geo` | PostgreSQL + PostGIS | geodata service | geometries, categories, provenance, imports, preprocessing, conflation, review and geodata outbox |

Local Compose runs three database containers. The Helm chart can either
provision three separate single-replica StatefulSets with persistent volume
claims or connect to externally managed PostgreSQL endpoints. The core and
activity instances use plain PostgreSQL; only geodata requires PostGIS. The
chart-managed database option is persistent but not highly available and needs
off-host backups.

The migration runner applies each service-owned migration set to its target.
During the first local split it copies activity rows from the old activity
tables in the existing `myota_core` database into an empty `myota_activity`
database. It also copies application geodata tables from the legacy
core-hosted `myota_geo` database into an empty PostGIS target. Copies are
guarded by source/target row-presence checks, so rerunning the job is
idempotent. The geodata copy includes service-local operational state (outbox,
idempotency, consumer checkpoints/processed events, dead letters and service
state), in addition to domain tables. Operators verify row counts in each
destination before retiring the legacy source. In the local Colima environment,
the copied operational rows were verified, duplicate activity tables were
removed from the old core database, and the old core-hosted `myota_geo` copy was
dropped; the active `myota_geo` PostGIS database remains authoritative.

## Consequences

- Core and activity avoid the PostGIS extension and spatial maintenance cost.
- Each database has its own `service_state`, idempotency, outbox, consumer
  checkpoint and dead-letter tables; these are service-local infrastructure,
  not shared control-plane tables.
- Activity QSO ingestion, worker pools and backups can scale independently of
  control-plane and geodata workloads.
- Cross-service relationships use opaque IDs, slugs and events rather than
  cross-database foreign keys.
- Production operators must back up and monitor three targets and provide all
  three database URL Secret keys.
- Migration copies are synchronized between the activity service, platform
  bootstrap and deployment repository; geodata synchronization remains governed
  by ADR-0006.
