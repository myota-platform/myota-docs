# Geodata import validation and promotion

This is the safety boundary for file and pasted-data imports. Pre-processing
normalizes and validates the source without making an unreviewed record part of
the public entity catalogue.

```mermaid
flowchart LR
  Source[File or pasted source] --> Intake[Import API]
  Intake --> Pre[Pre-processing worker]
  Pre --> Dedup[Duplicate verification]
  Dedup --> Store[(Pre-processed candidate store)]
  Dedup -.-> Warning[Possible duplicate warning]
  Store --> Admin[Visible pre-processing queue]
  Warning -.-> Admin
  Admin --> Validate[Confirmed selection]
  Validate --> NATS[NATS promotion subject]
  NATS --> Candidate[CANDIDATE entity]
  NATS --> Approved[APPROVED entity]
  Candidate --> Review[Geodata Review]
```

The candidate store is represented by `geodata_import_candidate` in the
geodata PostGIS schema. Duplicate verification compares normalized geometry
with existing entities; identical geometry or a centroid distance under 50
metres adds `POSSIBLE_DUPLICATE` and comparison geometry to the candidate. It
does not automatically merge, reject, or modify an existing entity. The admin
modal shows both geometries on a Leaflet map.

`geodata_import_processing_queue` is the durable projection of the NATS
request. The local development worker consumes the same logical queue directly
while production deployments connect the outbox to NATS JetStream. The
pre-processing queue remains separate from Geodata Review until the worker
materializes a confirmed record as an entity.

The target status is chosen only after confirmation. The imports collection
returns candidate counts so the admin page can keep active pre-processing runs
visible while completed runs move to import history. A candidate can later be
reviewed through the normal lifecycle (`CANDIDATE` to `APPROVED` or `REJECTED`),
while an import may use `APPROVED` only when the administrator has the required
review authority.
