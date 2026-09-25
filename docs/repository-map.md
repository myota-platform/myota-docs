# Proposed MyOTA repositories

The following split is justified and intentionally small:

| Repository | Owns | Initial source here |
|---|---|---|
| `myota-contracts` | OpenAPI, event schemas, compatibility rules, generated client release | `contracts/` |
| `myota-identity-service` | accounts, callsigns, auth claims, OIDC mappings | `services/identity.py`, core migrations |
| `myota-programme-service` | programmes, shared entity category master data and programme assignments, programme-owned rules and themes | `services/programmes.py`, core migrations |
| `myota-geodata-service` | PostGIS, import adapters, provenance, conflation, review, candidate-only dataset intake and source decoding | `services/geodata.py`, `import_formats.py`, geo migrations |
| `myota-activity-service` | activations, normalized/indexed QSOs, COPY/ADIF ingestion, activity aggregates, corrections, programme-owned award definitions and versioned progress, object-storage assets, requests, rendering, notifications, statistics and issuance records | `activity.py`, `awards.py`, `activity_repository.py`, `activity_worker.py`, `migrations/` |
| `myota-web` | universal programme UI, published award progress and participant requests | `web/` |
| `myota-admin-web` | authenticated global administration web, programme context, review queues, award designer, asset management and operational views | `myota-admin-web/web/` |
| `myota-deploy` | Helm charts, environments, migration orchestration, Compose, worker deployments, observability | `deploy/`, `db/migrations/`, `compose.yaml` |
| `myota-docs` | architecture, ADRs, operator and migration docs | `docs/` |

Migration ownership follows the service boundary. The activity repository is
the source of truth for `migrations/001_activity_relational.sql`, while
`myota-deploy` carries the deployment-applied copy as
`db/migrations/core/002_activity.sql` plus subsequent additive activity
migrations and runs them in release order. The
geodata repository is the source of truth for its complete ordered migration
set in `migrations/`; `myota-platform/db/migrations/geo/` is the synchronized
vertical-slice bootstrap mirror and `myota-deploy/db/migrations/geo/` is the
synchronized deployment mirror. The mirrors must not be edited independently.
Shared platform tables belong only to the core migration. This keeps schema
review close to the owning service without making every service perform
cluster migration orchestration.

The geodata import page is owned by `myota-admin-web`, but import semantics
remain owned by `myota-geodata-service`: text is parsed at the intake boundary,
uploads are malware-scanned and stored in MinIO/S3-compatible object storage,
and durable `geodata.import.queued.v1` events are published through the geo
outbox to NATS. Every dataset import requires a programme category and writes
`CANDIDATE` entities only. Supported intake formats are GeoJSON, KML, GPX,
WFS/ArcGIS GeoJSON, Shapefile archives, OSM PBF, and ParkServe US payloads;
PBF/ParkServe binary objects remain queued for the corresponding source worker.

Global entity deletion is an explicit cross-service workflow. The activity
service owns the impact calculation, valid-QSO deletion, aggregate rebuild and
award-progress recalculation job; only after that succeeds does the geodata
service remove the entity, conflation links and entity audit history. The UI
must show the QSO/activation impact and require confirmation before invoking
the irreversible operation.

The bootstrap repository is a temporary integration workspace; it is not a reason to create many more repositories. Each row is wired together by pinned contract versions.
