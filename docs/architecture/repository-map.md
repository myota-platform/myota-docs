# Current MyOTA repository map

The following split is justified and intentionally small:

| Repository | Owns | Current implementation / artifacts |
|---|---|---|
| `myota-contracts` | OpenAPI, event schemas, compatibility rules, generated client release | `contracts/` |
| `myota-identity-service` | accounts, callsigns, auth claims, OIDC mappings and assignable operations permissions | `identity.py`, `run_identity.py` |
| `myota-programme-service` | programmes, shared entity category master data and programme assignments, programme-owned rules and themes | `programmes.py`, `run_programmes.py` |
| `myota-geodata-service` | PostGIS, import adapters, provenance, conflation, review, staged dataset intake, source decoding and durable domain workers | `geodata.py`, `relational_state.py`, `relational_queries.py`, `geodata_import_worker.py`, `migrations/` |
| `myota-activity-service` | activations, normalized/indexed QSOs, COPY/ADIF ingestion, activity aggregates, corrections, programme-owned award definitions and versioned progress, object-storage assets, requests, rendering, notifications, statistics and issuance records | `activity.py`, `awards.py`, `activity_repository.py`, `activity_worker.py`, `migrations/` |
| `myota-operations-service` | Current authenticated NATS/JetStream and SeaweedFS inspection, sampled history, and current-identity Grafana role resolution; no business-domain queue processing. NATS-specific inspection is planned for retirement after Surveyor cutover. | `operations.py`, `storage_observability.py`, operations-owned core migrations |
| `myota-web` | universal programme UI, published award progress and participant requests | `web/` |
| `myota-admin-web` | Vue 3/TypeScript administration, programme context, grouped workspaces, resumable import intake, review queues, award designer, asset management, and operational views. The current JetStream status page is planned for retirement. | `src/`, `public/vendor/` |
| `myota-deploy` | Helm charts, environments, migration orchestration, Compose, worker deployments, observability | `deploy/helm/myota/`, `db/migrations/`, `compose.yaml` |
| `myota-docs` | architecture, ADRs, operator and migration docs | `docs/` |
| `myota-platform` | integration bootstrap and synchronized runtime, migration and contract mirrors; not domain ownership | `services/`, `db/migrations/`, `contracts/`, `tests/` |
| `.github` | public organization profile, delivery checklist and shared quality workflow | `profile/README.md`, `.github/workflows/python-quality.yml` |

Migration ownership follows the service boundary. The activity repository is
the source of truth for `migrations/001_activity_relational.sql`, while
`myota-deploy/db/migrations/activity/` and
`myota-platform/db/migrations/activity/` carry synchronized deployment and
bootstrap copies. The deployment runner applies these files to
`myota_activity`, never to `myota_core`. The
geodata repository is the source of truth for its complete ordered migration
set in `migrations/`; `myota-platform/db/migrations/geo/` is the synchronized
vertical-slice bootstrap mirror and `myota-deploy/db/migrations/geo/` is the
synchronized deployment mirror. The mirrors must not be edited independently.
Operational history is owned by `myota-operations-service/migrations/001_operations.sql`
and mirrored as core deployment migration `002_operations.sql`. Storage
snapshot history is owned by service migration `002_storage_snapshots.sql`
and mirrored as core migration `003_storage_snapshots.sql`. Geodata-local
idempotency, audit and outbox tables belong to its physical database; they are
not shared across database boundaries. This keeps schema
review close to the owning service without making every service perform
cluster migration orchestration.

The NATS monitoring consolidation plan in
[observability](../observability/nats-surveyor-migration.md) assigns Surveyor
and Grafana provisioning to Deploy, service monitoring retirement to each
owning service, and the synchronized deployment/bootstrap copies to Platform.
The existing Operations and Admin UI NATS inspection paths remain current until
that plan's rollout and overlap gates pass.

The physical storage split is intentional: `myota_core` and
`myota_activity` use plain PostgreSQL; only `myota_geo` uses PostGIS. Local
Compose runs three database containers. Helm expects three independent
database URLs in the `myota-postgres` secret (`core-database-url`,
`activity-database-url`, and `geo-database-url`).

The geodata import page is owned by `myota-admin-web`, but import semantics
remain owned by `myota-geodata-service`: pasted text is accepted through the
import resource, while files use a user-bound resumable session and bounded S3
multipart parts. The database stores upload metadata and per-part checksums;
the bytes are stored in SeaweedFS/S3-compatible object storage, not a shared
pod spool volume. The browser retries parts and can resume from the stored
part list. Object verification and malware gates run before the import run and
outbox event are accepted. Durable `geodata.import.queued.v1` events are
published through the geo outbox to NATS JetStream. The same geodata-owned
worker image runs separately from the HTTP Deployment and consumes durable
pull queues for preprocessing and administrator-requested promotion with
explicit ack, bounded redelivery, database leases/heartbeats, and idempotent
effects. File and pasted imports stop at `PREPROCESSED` as durable
`geodata_import_candidate` records. Identical geometry or a centroid distance
under 50 metres creates a non-blocking `POSSIBLE_DUPLICATE` warning with
comparison geometry. The admin web exposes these records in a dedicated
pre-processing queue with candidate counts; after paged confirmation, a
separate NATS processing queue promotes selected records to `CANDIDATE` or
authorized `APPROVED`. Only promoted candidates enter Geodata Review. Dataset
imports select one or more shared Master data entity categories, are not tied
to a programme, and programme assignment remains a separate eligibility
concern. Supported intake formats are GeoJSON, KML, GPX, WFS/ArcGIS GeoJSON,
Shapefile archives, OSM PBF, and ParkServe US payloads; PBF/ParkServe binary
objects remain queued for the corresponding source worker. The default
geodata request and upload envelopes default to 1 GiB via
`MYOTA_MAX_BODY_BYTES` and `MYOTA_UPLOAD_MAX_BYTES`; deployments can lower
both values. Browser file parts have a smaller configurable request-body cap;
the legacy full-file upload route is disabled when durable storage is on.

The geodata service also owns import retention. A daily worker expires
`PROCESSED` runs 30 days after `processed_at`; queued, preprocessed, failed, and
stalled runs expire after 30 days without a newer start/completion/heartbeat.
Active processing is retained while its heartbeat advances. The worker deletes
stored source files and import-specific history/log data, while promoted
entities and their source provenance remain.

Global entity deletion is an explicit cross-service workflow. The activity
service owns the impact calculation, valid-QSO deletion, aggregate rebuild and
award-progress recalculation job; only after that succeeds does the geodata
service remove the entity, conflation links and entity audit history. The UI
must show the QSO/activation impact and require confirmation before invoking
the irreversible operation.

Deletion is dispatched through the third geodata durable pull consumer,
`geodata-entity-deletion-v1`, on `myota.geodata.entity.delete.v1`. The job
stores verified authorization context, leases execution and uses idempotent
activity API calls; it is not an API-local executor. Preprocessing uses
`geodata-preprocessing-v1`; promotion uses `geodata-import-processing-v2`;
location enrichment uses `geodata-location-enrichment-v1` after entity
materialization or geometry updates. Provider calls are asynchronous, guarded by
request ID/geometry hash, and do not overwrite manual values.
The operations service inspects these consumers but never executes their work.
See the [Phase 1 authority/rollout record](../geodata/phase1-relational-authority.md)
and [latest scaling delivery evidence](../geodata/horizontal-scaling-roadmap.md#latest-delivery-and-evidence--8-october-2026).

The bootstrap repository is a temporary integration workspace; it is not a
reason to create many more repositories. Each row is wired together by pinned
contract versions. The public participant experience belongs in `myota-web`;
administrator-only workflows belong in `myota-admin-web`. Android and iOS,
when implemented, are participant applications only and must not become an
alternate administration surface.

## Documentation synchronization rule

`myota-docs` is the authoritative home for cross-repository architecture,
ADRs, ownership boundaries, operational guidance, and gap analysis. The
organization `.github` profile is the concise public summary and checkbox
roadmap. Service repositories keep implementation-specific README and API
notes, but must link back here for cross-service claims.

When a service boundary, API, migration owner, deployment topology, storage
provider, or user-facing capability changes:

1. update the owning service README/API notes;
2. update this repository map and the relevant central architecture/ADR;
3. update `.github/profile/README.md` if the public summary or roadmap status
   changed; and
4. update diagrams and deployment/operator docs when topology or lifecycle
   changed.

Documentation changes should be committed with the implementation change or
as an immediately following documentation commit. A repository README must
not claim a capability is production-ready merely because a dependency-free
test adapter or local vertical slice exists.
