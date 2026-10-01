# MyOTA platform architecture

Editable Mermaid views are maintained in [`diagrams/service-boundaries.md`](diagrams/service-boundaries.md), [`diagrams/programme-configuration-lifecycle.md`](diagrams/programme-configuration-lifecycle.md), and [`diagrams/data-model.md`](diagrams/data-model.md). This document remains the narrative architecture reference; the diagrams intentionally show the major ownership and lifecycle relationships without replacing detailed API or migration documentation.

## Scope

MyOTA is an Outdoor Activation Platform. A programme is configuration and policy data consumed by platform capabilities. MPOTA is only sample seed data; future programmes use the same APIs without cloning a codebase. The platform does not copy, inherit or silently normalize another programme's charter, rules, minimum QSOs, award logic or eligibility policy. Those are programme-owned inputs, versioned and auditable as configuration or programme code.

The product motivation and working charter are documented in
[`project-charter.md`](project-charter.md). The current architecture is a
meaningful local vertical slice, not a claim that the participant experience,
community governance, public Explorer, or Internet-facing operational gates
are complete; see [`charter-gap-analysis.md`](charter-gap-analysis.md).

## Service boundaries

```mermaid
flowchart LR
  UI[Universal web frontend] --> G[API gateway / ingress]
  G --> I[Identity service\naccounts, callsigns, roles]
  G --> P[Programme service\nconfiguration, rules, themes]
  G --> Geo[Geodata service\nPostGIS, imports, review]
  G --> A[Activity service\nactivations, QSOs, awards]
  I -. events .-> Bus[(Event broker / outbox)]
  P -. events .-> Bus
  Geo -. events .-> Bus
  A -. events .-> Bus
  I --> C[(myota_core)]
  P --> C
  A --> C
  Geo --> D[(myota_geo\nPostGIS)]
  Admin[QGIS / browser map editor] --> Geo
```

Each service owns its database tables and publishes events. No service reads another service's tables. The gateway/ingress is a routing boundary, not a domain owner.

Activity execution and award management are one bounded service and one API
deployment. Locally, both route families are exposed on port `8004`; award
paths remain distinct (`/v1/awards`) but do not create a second service or
port. The activity service owns normalized activation, QSO, aggregate, award,
import, correction, job, statistic and notification tables. It does not use
the generic JSONB `service_state` projection in durable mode. Activity and
award writes use bounded psycopg pools and transaction-local idempotency.

Geodata uses the same compatibility snapshot during the migration period, but
the authoritative entity catalogue is also upserted into the PostGIS-owned
`geodata_entity`, `geodata_entity_category`, `source_reference`, and review
tables. Manual candidates and their lifecycle changes therefore remain durable
independently of the JSON snapshot and are visible to QGIS. Hydration merges
relational-only entities and overlays category assignments onto legacy
snapshot entities; only an explicit API deletion removes a relational entity.

The service exposes two ingestion paths: ordinary idempotent QSO writes and a
PostgreSQL `COPY` staging path for batches and ADIF worker output. Distinct
subject/callsign/entity membership tables support precomputed activator and
hunter facts, while `activity_award_progress` stores a versioned evaluation
for historical award definitions. Reads never scan the full QSO table to
render participant progress.

The API deployment is stateless and horizontally scalable. Helm defaults to
three activity replicas; `activity_worker` handles ADIF parsing, award-rule
recalculation, PDF rendering, statistics and notification delivery, while the
notification consumer translates geodata and identity events into participant
notices. Each workload has its own bounded database pool.

Award background images, manager signatures and generated certificate objects
are addressed through an S3-compatible object store. Local Compose provides
SeaweedFS; production Helm values point the activity service at the selected
managed or self-hosted object store. Metadata and immutable issuance render
specifications remain in the activity service so object storage can be replaced
without changing the API.

## Geodata lifecycle

```mermaid
flowchart TD
  S[Authoritative/imported source] --> R[Adapter + import run]
  R --> P[Pre-process and normalize]
  P --> D[Duplicate verification]
  D --> Store[Pre-processed candidate store]
  D -.-> W[Possible duplicate warning]
  Store --> V[Admin validation queue]
  W -. review in queue .-> V
  V --> Q[NATS promotion queue]
  Q --> C[CANDIDATE]
  Q --> A[APPROVED]
  Community[Community proposal] --> C
  C -->|approver scope + review| A
  C -->|approver decision| X[REJECTED]
  A --> M[Public map + activation eligibility]
```

The UI distinguishes the **pre-processing queue** from **Geodata Review**. An
import run first reaches `QUEUED`, `PROCESSING`, and then `PREPROCESSED` (or
`PREPROCESSED_WITH_ERRORS`); its normalized records are stored as
`geodata_import_candidate` rows and are not entities or programme references
yet. Pre-processing isolates failures per feature: successfully normalized
records remain staged for validation, while failed feature indexes and messages
are retained in the run summary and only those records are omitted from the
candidate queue. During pre-processing each record is
checked against existing entity geometry. Identical geometry or a centroid
distance below 50 metres produces `dedupeWarning=POSSIBLE_DUPLICATE` and
comparison geometry/details. This warning is deliberately non-blocking: an
administrator reviews it in the import queue's map modal and decides whether
to confirm, reject, or leave the record pending. The admin imports page keeps
these runs in a visible pre-processing queue, separate from completed import
history and Geodata Review.

Candidate provenance records whether the record came from an `ADAPTER_IMPORT`
(with adapter/import-run identifiers) or a `COMMUNITY_PROPOSAL` (with
proposal/proposer identifiers). After selection and confirmation, the admin
publishes a durable NATS processing request. Only that queue worker materializes
the selected record as `CANDIDATE` or authorized `APPROVED`; a resulting
`CANDIDATE` then enters normal Geodata Review. Import refreshes update source
provenance and geometry while preserving review state; an explicit policy can
retire records that disappear from an authoritative source. Supported GeoJSON
geometry types are `Point`, `LineString`, `MultiLineString`, `Polygon`, and
`MultiPolygon`; trails/routes commonly use `LineString` (called a `way` in the
administration UI). The importer also accepts the non-standard `way` geometry
alias and normalizes it to `LineString`. Entity categories such as
`MUNICIPAL_PARK` and `TRAIL` are shared master-data definitions that can be
assigned to multiple programmes and can allow more than one geometry type;
review changes are made through the geodata API and recorded in entity history.

## Import adapters

The adapter interface is a normalized feature stream:

```text
discover(source_config) -> source snapshot metadata
read(snapshot) -> {source_record_id, name, geometry, properties, license}
normalize(feature) -> canonical geometry/properties
conflate(feature, existing) -> match candidates + score
apply(feature, policy) -> candidate/update/retire
```

Required adapters are represented in the contract and storage model: `PARKSERVE_US`, `OSM`, `GOVERNMENT_GIS`, and `MANUAL`. Intake accepts GeoJSON, KML, GPX, WFS/ArcGIS GeoJSON, Shapefile archives (`.shp` with `.shx`/`.dbf` sidecars), OSM PBF, and ParkServe US binary payloads. ParkServe and government feeds remain source-specific integrations; OSM imports preserve ODbL attribution and retrieval metadata. Manual community proposals use the same entity/review path and do not bypass approval. They are a candidate source alongside adapter/import runs, not a separate lifecycle state. Dataset imports select one or more shared Master data categories and are programme-independent. File and pasted imports first create durable `geodata_import_candidate` records and stop at `PREPROCESSED` or `PREPROCESSED_WITH_ERRORS`; duplicate verification is part of this step, not a silent merge. A feature-level preprocessing failure does not discard the rest of the import: valid records remain pending and failed indexes/messages remain in the run summary. An administrator validates a paged selection in the separate pre-processing queue, then publishes a processing request to the `myota.geodata.import.process.v1` NATS subject with target `CANDIDATE` or `APPROVED`. Only that worker creates or updates `geodata_entity`; programme assignment is handled separately.

The import safety boundary is explicit in both the API and schema. `GET
/v1/geodata/imports` returns import-run status plus pending, confirmed,
processed, and rejected candidate counts for the visible pre-processing queue.
`GET /v1/geodata/imports/{runId}/candidates` returns compact pages of pending
records only. Confirmed, rejected, and processed staging records are omitted
from the detail queue; rejected records are deleted immediately and successful
promotion deletes the staged record. `POST .../candidates/validate` records
the administrator confirmation or rejection. `POST
/v1/geodata/imports/{runId}/process` creates a durable processing-queue record
and an outbox event. Local development uses the same bounded worker as a
fallback; production NATS consumers use the event payload and are idempotent.
Records must be `CONFIRMED` before promotion, and the selected target is
restricted to `CANDIDATE` or `APPROVED`. The PostGIS geometry and full normalized
payload remain available for validation without exposing an unconfirmed record
as a live entity.

After validation and any desired promotion, an administrator can finalize the
run with `POST /v1/geodata/imports/{runId}/processed`. This is an explicit,
idempotent cleanup action: it deletes that run's staged candidate rows and
promotion-queue rows, records the actor and timestamp, changes the run to
`PROCESSED`, and preserves the top-level import summary for audit and history.
It does not delete entities that were already materialized by the promotion
worker. The import UI hides the validation queue after finalization and keeps
only that summary.

Import execution is restart-safe. Before a queued background run starts, its
uploaded or pasted source is stored in SeaweedFS and the `import_run` row is
claimed with a PostgreSQL lease. The worker refreshes `heartbeat_at` and
`lease_until` while it parses, normalizes, reverse-geocodes and persists
candidates. On service startup, queued runs and all runs left in
`PROCESSING` by the previous instance are requeued from their immutable
object-storage source, and the requeue is persisted before workers are
dispatched. Pasted KML/GPX is replayed from the normalized GeoJSON snapshot;
uploaded files retain their original parser format. Durable binary uploads
whose parser adapter is not available remain queued with a visible status
instead of being incorrectly failed. If the source is missing or was never
durably recorded, the run is marked `FAILED` with a visible recovery error
rather than remaining indefinitely in `PROCESSING`. The default lease is
15 minutes; operators can tune it with `MYOTA_IMPORT_LEASE_SECONDS` and
the heartbeat interval with `MYOTA_IMPORT_HEARTBEAT_SECONDS`.

### Shared category selection and persistence

```mermaid
sequenceDiagram
  participant Admin as Admin web
  participant Programme as Programme service
  participant Geo as Geodata service
  participant Entity as geodata_entity
  participant Assignment as geodata_entity_category

  Admin->>Programme: GET /v1/entity-types
  Programme-->>Admin: active shared category catalogue
  Admin->>Geo: proposal/import with entityTypes[]
  Geo->>Geo: normalize, deduplicate, choose first as entityType
  Geo->>Entity: upsert geometry and primary compatibility code
  Geo->>Assignment: replace all category assignments
  Assignment-->>Geo: primary + additional categories durable
  Geo-->>Admin: entityType + entityTypes + entityTypeCodes
```

The browser never owns the category catalogue. The first category in the
ordered request is retained as the singular compatibility value, while
`geodata_entity_category` is authoritative for the complete assignment. The
same relation supports entities with no programme assignment and categories
assigned to several programmes.

The administration web has a dedicated Geodata imports page. It loads every category from the database-backed shared `/v1/entity-types` catalogue rather than a programme-scoped or hardcoded list. It supports copy/paste for text documents and file upload for binary or text documents, including an explicit OpenStreetMap GeoJSON option that routes GeoJSON through the OSM tag adapter. Browser file uploads use `multipart/form-data` with the metadata JSON and original file as separate parts; the legacy JSON/Base64 representation remains available for non-browser clients. This avoids constructing a potentially oversized in-memory Base64 string in the browser. Upload requests are capped at 1 GiB by default and the gateway disables request buffering for them; the geodata service spools multipart files to a temporary file, scans them incrementally, and streams that file to SeaweedFS with a single S3 `PutObject` request. This avoids multipart-upload finalization stalls with SeaweedFS while preserving bounded memory use. Uploads pass a size/malware gate, are stored under the geodata-import bucket, and generate an outbox event for NATS processing. The synchronous local decoder covers GeoJSON, KML, GPX and Shapefile archives; OSM PBF and ParkServe binary records are retained as queued source objects for their adapter workers. During pre-processing, each normalized record is checked against existing entity geometry; identical geometry or a centroid distance below 50 metres produces a `POSSIBLE_DUPLICATE` warning with comparison geometry, while the administrator retains the final decision. The imports collection exposes pending/confirmed/processed candidate counts, allowing the admin web to display a dedicated pre-processing queue before records are explicitly promoted into Geodata Review. Compose uses SeaweedFS through its S3-compatible API; production Helm deployments can use the optional single-node SeaweedFS chart or an externally operated SeaweedFS endpoint with S3 credentials.

Import normalization also derives a canonical display name before a record enters the preprocessing queue. An explicit `name` wins, followed by common source aliases such as `SITE_NAME`, `official_name`, `NOMBRE`, `DENOMINACION`, `title`, and `label`; source codes, identifiers, geometry fields, and administrative metadata are excluded from name inference. The original property map remains unchanged in provenance, so the derived `name` is a review-friendly projection rather than a loss of source data. Records without a usable alias retain the `Unnamed candidate` fallback.

The administration web also provides a read-only **Entity map** page. It loads
the complete paged entity catalogue, renders all geometries in Leaflet with
lifecycle-specific colours, clusters entity location markers with
Leaflet.markercluster, and opens a popup containing the entity name, location
metadata, shared categories, and derived programme memberships. Line and
polygon geometries remain visible as selectable overlays. The page is
intentionally separate from Geodata Review: it never enables geometry editing
or changes review state. Programme membership is derived from explicit
entity assignment when present and from the shared category assignments held by
the programme service, so unassigned entities remain visible.

## Security and operations

- Short-lived access tokens are verified at the gateway; the identity service issues radio-native claims (`account_id`, participation type, verified callsigns, scopes).
- OAuth/OIDC is optional per programme configuration and is never the identity source of truth. External subject mappings point to an internal account.
- Approver authorization is scope-based: programme + jurisdiction + entity type. Review mutations require an approver scope and are audit events. Shared category changes and display-name corrections are allowed for platform-wide entities as well as programme-assigned entities and preserve the previous value in audit history. Global and GIS administrators may convert point/way/polygon geometry with an audit record. GIS administrators may permanently delete rejected entities; a global administrator may delete any status. The global deletion workflow first calculates impact in the activity service, deletes linked valid QSOs, rebuilds aggregates and queues award recalculation, then removes the entity, conflation links and its audit record.
- The admin review queue is catalogue-driven rather than viewport-driven. Programme (including unassigned entities), entity type, continent, country, region/subdivision, province, city/municipality, and multi-select lifecycle status filters are applied by geodata before deterministic pagination. The map renders the current page and remains independently pannable/zoomable; selecting a result centres it without changing the queue.
- Every mutation accepts `Idempotency-Key`; service outboxes make event publication retry-safe.
- Rate limits apply at gateway, with stricter limits for import and proposal endpoints.
- JSON logs carry request, correlation, actor and programme IDs. Health/readiness endpoints are available per service.

## Activity execution boundary

Activation start/close captures a programme-rule snapshot and evaluates
validity windows, location requirements, operator/callsign authorization,
minimum QSOs, band/mode rules and correction state. Programme policy remains
owned by the programme service; the snapshot is retained so a later policy
change cannot silently rewrite a historical activation decision.

ADIF uploads are stored as objects after size and malware gates, then processed
as durable jobs. Parsing and normalization are separate from programme policy:
the worker validates callsign, timestamp, band, mode and worked-station
identity before the activity repository applies deduplication and aggregate
updates. Corrections are proposed and reviewed rather than mutating history
without an audit trail.
