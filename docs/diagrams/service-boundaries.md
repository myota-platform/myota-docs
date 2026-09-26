# MyOTA service and data ownership

This diagram is the practical ownership view for the current repository split.
The admin web is a control plane and does not own domain data. Cross-service
references are exchanged as IDs, slugs, and versioned API/event payloads; they
are not cross-database foreign keys.

```mermaid
flowchart LR
  Participant[Participant web / Android / iOS]
  Admin[Admin web]
  Gateway[Ingress / API gateway]
  Events[(NATS / durable outbox events)]
  Core[(Core PostgreSQL schema)]
  GeoDB[(Geodata PostgreSQL + PostGIS)]
  ActivityDB[(Activity PostgreSQL schema)]
  Objects[(SeaweedFS / S3 object storage)]

  Participant --> Gateway
  Admin --> Gateway
  Gateway --> Identity[Identity service\naccounts, callsigns, roles, OIDC]
  Gateway --> Programme[Programme service\nprogramme config, policies, locales, jurisdictions]
  Gateway --> Geodata[Geodata service\nentities, imports, provenance, review]
  Gateway --> Activity[Activity service\nactivations, QSOs, awards, statistics]

  Identity --> Core
  Programme --> Core
  Geodata --> GeoDB
  Activity --> ActivityDB
  Geodata --> Objects
  Activity --> Objects

  Identity -. publishes/consumes .-> Events
  Programme -. publishes/consumes .-> Events
  Geodata -. publishes/consumes .-> Events
  Activity -. publishes/consumes .-> Events
  Events -. notifications, recalculation, imports .-> Workers[Background workers]
  Workers --> Activity
  Workers --> Geodata
```

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
