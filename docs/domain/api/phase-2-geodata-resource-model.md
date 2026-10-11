# Phase 2 — Geodata resource model

**Status:** complete  
**Completed:** 2026-10-11  
**Owner:** myota-geodata-service; OpenAPI is owned by myota-contracts.

The Geodata API exposes imports, proposals, entities, and deletion jobs as
resources. The canonical operation schemas are in
[OpenAPI](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).
The Deploy and Platform route copies are checked against the service owner.

## Current operations

| Resource | Operation | Purpose |
| --- | --- | --- |
| Import | **POST /v1/geodata/imports** | Accept an import request and create durable preprocessing work. |
| Upload session | **POST /v1/geodata/import-uploads** | Create an owner-bound, checksummed upload session. |
| Upload part | **POST /v1/geodata/import-uploads/{uploadId}/parts/{partNumber}** | Upload a bounded binary part with an independent checksum. |
| Upload completion | **POST /v1/geodata/import-uploads/{uploadId}/complete** | Verify the assembled object and create the import run. |
| Proposal | **POST /v1/geodata/proposals** | Create a community or manual candidate. |
| Entity metadata | **PATCH /v1/geodata/entities/{entityId}** | Update name and location metadata while preserving manual-field precedence. |
| Entity categories | **PUT /v1/geodata/entities/{entityId}/categories** | Replace the shared category set atomically. |
| Entity geometry | **PUT /v1/geodata/entities/{entityId}/geometry** | Replace validated GeoJSON and append geometry history. |
| Entity review | **POST /v1/geodata/entities/{entityId}/reviews** | Record the audited review decision. |
| Entity collection | **GET /v1/geodata/entities?bbox=minLon,minLat,maxLon,maxLat** | Query the paged collection by extent and supported filters. |
| Deletion | **POST /v1/geodata/entity-deletion-jobs** | Calculate linked Activity impact and create a confirmation boundary. |
| Deletion status | **GET /v1/geodata/entity-deletion-jobs/{jobId}** | Read durable job state and impact. |
| Deletion confirmation | **POST /v1/geodata/entity-deletion-jobs/{jobId}/confirm** | Confirm the cascade before entity and audit-history removal. |

Imports use one durable staged lifecycle. Preprocessed records remain separate
from catalogue entities until an administrator validates and promotes them.
Partial failures remain visible on the import run; they do not silently discard
valid records. Activity owns its cascade and award recalculation work, while
Geodata owns the entity and review records.

File uploads use the resumable session flow; pasted JSON remains an import
request. Import bytes are kept in SeaweedFS/S3-compatible object storage rather
than pod-local disk. The browser can retry and resume bounded parts.

## Module and boundary decisions

The service now registers routes in
[geodata_routes.py](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata_routes.py).
Import processing, adapters, relational state, and worker responsibilities
remain in their existing Geodata-owned modules. No new service or database was
introduced. This keeps authorization, transaction, outbox, idempotency, and
recovery ownership in Geodata.

## Evidence

- The source-to-mirror audit compares the Geodata API modules and both runtime
  copies.
- Contracts CI reports zero missing or unregistered API operations.
- Geodata Python quality and image-build workflows pass on current main.
- Focused upload, import, deletion, and worker regression evidence remains
  linked from the [Geodata evidence index](../../geodata/README.md).

See the [REST API consolidation record](rest-consolidation-plan.md) and the
[cross-repository phase evidence](../../architecture/evidence/review-phases-1-4-2026-10-11.md).
