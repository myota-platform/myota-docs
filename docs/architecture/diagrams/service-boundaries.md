# MyOTA service and data ownership

This current-state view summarizes service boundaries. The admin web
coordinates APIs but does not own domain data. Cross-service references are
IDs, slugs, and versioned API/event payloads, never cross-database foreign keys.

```mermaid
flowchart LR
  Participant[Participant web and mobile]
  Admin[Admin web]
  Gateway[Ingress and API gateway]
  Identity[Identity service]
  Programme[Programme service]
  Activity[Activity API and workers]
  Geodata[Geodata API and workers]
  Core[(myota_core)]
  ActivityDB[(myota_activity)]
  GeoDB[(myota_geo and PostGIS)]
  Objects[(SeaweedFS S3)]
  Events[(MYOTA_EVENTS<br/>bounded fact stream)]
  ActivityWork[(MYOTA_ACTIVITY_WORK<br/>competing-consumer jobs)]
  GeoWork[(MYOTA_GEODATA_WORK<br/>competing-consumer jobs)]
  Surveyor[NATS Surveyor]
  Prometheus[(Prometheus)]
  Collector[OpenTelemetry Collector]
  Grafana[Grafana]

  Participant --> Gateway
  Admin --> Gateway
  Gateway --> Identity
  Gateway --> Programme
  Gateway --> Activity
  Gateway --> Geodata

  Identity --> Core
  Programme --> Core
  Activity --> ActivityDB
  Geodata --> GeoDB
  Geodata --> Objects
  Activity --> Objects

  Identity -. committed facts .-> Events
  Programme -. committed facts .-> Events
  Activity -. committed facts .-> Events
  Geodata -. committed facts .-> Events
  Activity --> ActivityWork
  Geodata --> GeoWork
  ActivityWork --> Activity
  GeoWork --> Geodata
  Events --> Activity

  Surveyor --> Prometheus
  Core -->|read-only database telemetry| Collector
  ActivityDB -->|read-only database telemetry| Collector
  GeoDB -->|read-only database telemetry| Collector
  Gateway -. live Identity session and role .-> Identity
  Collector --> Prometheus
  Grafana --> Prometheus
```

NATS is cluster-internal. Surveyor is the broker observer; Operations has no
NATS connection. Operations is retired. Deploy's Collector reads bounded database statistics, and Gateway checks current Identity session and role data before Grafana access. SeaweedFS monitoring flows from its
private metrics listener through the Collector to Prometheus and Grafana;
Operations does not sample storage or retain its history. PostgreSQL remains
authoritative for domain state, job state, outbox delivery, and recovery.

## Ownership

- Identity owns accounts, callsigns, authentication sessions, roles, scopes,
  and OIDC provider mappings.
- Programme owns programme metadata, policy versions, locale and jurisdiction
  configuration, and category catalogue/assignment APIs.
- Activity owns activations, QSOs, corrections, aggregates, award definitions,
  progress, issuance, and activity notifications.
- Geodata owns geometries, entity lifecycle, imports, provenance, conflation,
  review, and geodata audit history.
- Deploy owns Collector configuration; Gateway and Identity own Grafana
  session and role checks. Repository archival remains pending. It no longer samples SeaweedFS and does not inspect or
  consume JetStream.
- Deploy owns runtime wiring and deployment configuration. Platform copies of
  Deploy assets are synchronized mirrors; service repositories remain
  authoritative for service code and contracts.

For the latest live deployment revision and evidence limits, see
[Current system state](../current-state.md). The
[NATS diagram](nats-event-migration.md) distinguishes the current topology from
historical rollout snapshots.
