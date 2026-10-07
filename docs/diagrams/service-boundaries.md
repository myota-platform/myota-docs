# MyOTA service and data ownership

This diagram is the practical ownership view for the current repository split.
The admin web is a control plane and does not own domain data. Cross-service
references are exchanged as IDs, slugs, and versioned API/event payloads; they
are not cross-database foreign keys.

```mermaid
flowchart LR
  Participant[Participant web / Android / iOS]
  Admin[Admin web]
  Grafana[Grafana dashboards]
  Prometheus[(Prometheus)]
  Alertmanager[Alertmanager]
  Gateway[Ingress / API gateway]
  Events[(NATS / durable outbox events)]
  Core[(myota_core\nPostgreSQL)]
  ActivityDB[(myota_activity\nPostgreSQL)]
  GeoDB[(myota_geo\nPostgreSQL + PostGIS)]
  Objects[(SeaweedFS / S3 object storage)]

  Participant --> Gateway
  Admin --> Gateway
  Admin -->|JWT-checked /observability proxy| Grafana
  Admin -. auth subrequest .-> Gateway
  Grafana --> Prometheus
  Grafana --> Alertmanager
  Gateway --> Identity[Identity service\naccounts, callsigns, roles, OIDC]
  Gateway --> Programme[Programme service\nprogramme config, policies, locales, jurisdictions]
  Gateway --> Geodata[Geodata service\nentities, imports, provenance, review]
  Gateway --> Activity[Activity service\nactivations, QSOs, awards, statistics]
  Gateway --> Operations[Operations service\nJetStream status and sampled history]

  Identity --> Core
  Programme --> Core
  Operations --> Core
  Operations -->|Read-only broker inspection| Events
  Geodata --> GeoDB
  Activity --> ActivityDB
  Geodata --> Objects
  Activity --> Objects

  Identity -. publishes/consumes .-> Events
  Programme -. publishes/consumes .-> Events
  Geodata -. publishes/consumes .-> Events
  Activity -. publishes/consumes .-> Events
  Events -. geodata jobs .-> GeoWorkers[Geodata-owned workers]
  GeoWorkers --> GeoDB
  GeoWorkers --> Objects
  GeoWorkers -->|Deletion cascade API| Activity
  ActivityWorkers[Activity-owned execution and notification workers] --> ActivityDB
  Events -. activity events .-> ActivityWorkers
```

The runtime uses three database containers in local development and three
independently addressable database targets in Kubernetes. `myota_core` holds
identity, programmes, configuration, and shared control-plane data;
`myota_activity` holds activations, QSOs, aggregates, awards, jobs and
notifications; `myota_geo` holds PostGIS geometry, imports, provenance and
review data. Only `myota_geo` requires PostGIS. Cross-service references are
IDs, slugs and events rather than database foreign keys.

## Ownership rules

- `myota-programme-service` owns programme metadata, published policy
  versions, locale and jurisdiction configuration, and the shared category
  catalogue/assignment APIs.
- `myota-identity-service` owns accounts, callsigns, authentication sessions,
  roles, scopes, and OIDC provider mappings.
- `myota-geodata-service` owns geometries, entity lifecycle, imports, source
  provenance, conflation, review, and geodata audit history.
- `myota-activity-service` owns activations, normalized QSOs, corrections,
  aggregates, award definitions/progress/issuance, statistics, and activity
  notifications. Activity and awards share one API deployment.
- `myota-admin-web` coordinates these APIs; it does not write database tables
  directly.
- `myota-operations-service` owns read-only stream/consumer inspection and
  sampled history in `myota_core`; it never consumes business deliveries.
- Geodata-owned workers execute preprocessing, promotion and confirmed
  deletion with separate durable pull consumers and recoverable database
  leases. Activity workers remain in the activity domain. See the
  [latest delivery evidence and remaining scaling gates](../geodata-horizontal-scaling-roadmap.md#latest-delivery-and-evidence--7-october-2026).
- `myota-deploy` owns runtime wiring, migration orchestration, workers,
  secrets, object storage, NATS, and Kubernetes/Compose configuration.

## Request and event flow

```mermaid
sequenceDiagram
  participant U as User or admin
  participant W as Web/mobile client
  participant G as Gateway
  participant S as Owning service
  participant DB as Service database
  participant O as Outbox/NATS
  participant B as Worker

  U->>W: Submit API request
  W->>G: Versioned API call + access token
  G->>S: Authenticated request
  S->>DB: Transaction + idempotency check
  DB-->>S: Durable result
  S->>O: Transactional domain event
  S-->>G: API response
  G-->>W: Result
  O->>B: Retry-safe event delivery
  B->>S: Recalculation, import, PDF, or notification job
```
