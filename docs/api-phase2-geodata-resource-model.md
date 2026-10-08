# Phase 2 geodata resource model

Phase 2 publishes the geodata resource-oriented API while keeping the existing
routes available as compatibility aliases. The canonical contract is
[`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml);
the deployed geodata registry is checked against it by the contract-freeze
workflow.

## Preferred resources

| Resource | Preferred operation | Purpose |
| --- | --- | --- |
| Import | `POST /v1/geodata/imports` | Accept JSON features/text, multipart files, or a durable object reference and enqueue preprocessing. |
| Proposal | `POST /v1/geodata/proposals` | Create a community/manual candidate proposal. |
| Entity metadata | `PATCH /v1/geodata/entities/{entityId}` | Partially update the name and location metadata while preserving manual-field precedence. |
| Entity categories | `PUT /v1/geodata/entities/{entityId}/categories` | Atomically replace the shared category set; the first category remains the compatibility primary. |
| Entity geometry | `PUT /v1/geodata/entities/{entityId}/geometry` | Replace validated GeoJSON geometry and append geometry history. |
| Entity review | `POST /v1/geodata/entities/{entityId}/reviews` | Record the audited lifecycle decision. Approved entities retain the approved-to-retired invariant. |
| Entity collection | `GET /v1/geodata/entities?bbox=minLon,minLat,maxLon,maxLat` | Apply extent, status, category, programme, and location filters to one paged collection. Tile delivery remains a separate concern. |
| Deletion job | `POST /v1/geodata/entity-deletion-jobs` | Calculate QSO/activation/award impact and create an explicit confirmation boundary. |
| Deletion status | `GET /v1/geodata/entity-deletion-jobs/{jobId}` | Read impact and asynchronous execution status. |
| Deletion confirmation | `POST /v1/geodata/entity-deletion-jobs/{jobId}/confirm` | Require the literal `DELETE` confirmation, cascade activity data, then delete the geodata entity and audit history. |

## Import representations and lifecycle

The import collection is programme-independent. The request selects one or
more shared entity categories and includes provenance/source licensing metadata.
It supports:

- JSON GeoJSON features or copied text for GeoJSON, KML, GPX, WFS, and ArcGIS
  JSON;
- multipart upload for text and binary formats;
- an object-storage reference (`bucket`, `objectKey`, and optional checksum) for
  files already durably stored by an upload gateway.

Every representation enters the same durable staged lifecycle:

```text
POST /imports
  -> QUEUED / PROCESSING
  -> PREPROCESSED or PREPROCESSED_WITH_ERRORS
  -> candidate validation
  -> promotion queue (CANDIDATE or APPROVED)
  -> materialized entity
  -> explicit import finalization
```

`PREPROCESSED_WITH_ERRORS` is a partial-success state, not a discarded import.
Each feature is normalized independently. Valid features remain as pending
staging records for administrator validation, while failed feature indexes and
messages are retained in the import-run error summary and those failed features
are excluded from the validation queue.

Preprocessed records are not catalogue entities. Rejected, confirmed, and
processed staging records are removed according to the existing queue
semantics; the import summary remains available. All promoted records have
`CANDIDATE` or `APPROVED` status explicitly selected by the administrator.

## Compatibility policy

The following aliases remain registered and call the same service methods:

- `POST /imports/manual` and `POST /imports/upload` → `POST /imports`;
- `POST /proposals/draw` → `POST /proposals`;
- `POST /entities/{id}/review` and `/status` → entity reviews;
- `POST /entities/{id}/name` and `/location` → metadata patch;
- `POST /entities/{id}/entity-type` → categories replacement;
- `POST /entities/{id}/geometry` → geometry replacement;
- `GET /geodata/bbox` → the entity collection extent filter;
- `POST /entities/{id}/delete` → the deletion-job workflow during client
  migration.

Aliases emit `Deprecation: true` and
`Sunset: 2027-04-01T00:00:00Z`. They are not removed until client migration,
traffic review, and a later contract decision are complete. The preferred
resource operations do not emit those headers.

## Cross-service deletion safety

The geodata deletion job owns the user-facing confirmation boundary. During
job creation it calls the activity service through the preferred resource
endpoints `GET /v1/activations/entity-deletion-impacts/{entityId}` and
`POST /v1/activations/entity-deletion-cascades`, rather than the deprecated
`/admin/entities/.../deletion-impact` and `/cascade-delete` action routes.
The cascade request creates the activity-owned QSO deletion and award
recalculation work after the geodata job has been explicitly confirmed.

Deletion is deliberately a resource/job, not a single destructive request. The
creation response exposes the impact returned by the activity service. A
separate confirmation request is required. Confirmation queues the activity
cascade, which removes affected QSOs and recalculates derived award progress;
only after that succeeds does geodata remove the entity and its audit records.
Failures remain visible on the job and do not silently report success. Clients
should read `GET /v1/geodata/entity-deletion-jobs/{jobId}` after confirmation
and distinguish `QUEUED`/`PROCESSING` from terminal `COMPLETED`/`FAILED` states.
Bulk clients should preserve the selected jobs through partial failure and must
not treat an accepted `202` response as proof that deletion has completed.

## Verification

The owner service has direct tests for the review, metadata, categories, bbox,
and confirmation workflows. The contract mirror and runtime route registry are
checked with:

```text
python3 scripts/sync_contract_mirrors.py --platform-root ../myota-platform
python3 scripts/check_contract_phase0.py \
  --canonical contracts/openapi.yaml \
  --mirror openapi.yaml \
  --mirror ../myota-platform/contracts/openapi.yaml \
  --service-root ../myota-deploy/services \
  --semantic-baseline contracts/semantic-duplicates.json \
  --inventory-out contracts/route-inventory.json
```

See the [geodata resource lifecycle diagram](diagrams/geodata-resource-lifecycle.md)
for the request, queue, review, and deletion boundaries.
