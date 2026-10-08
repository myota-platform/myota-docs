# Entity catalogue editor

Implemented in `myota-admin-web` at `/entity-management` (9 October 2026).
This changes presentation and navigation only; the geodata service remains
the sole authority for permissions, persistence, geometry validation and deletion.

## Why the layout changed

Previously the shared review/management view stacked filters, 25–100 list
rows, a 680px map, source comparison, every editor and audit history. Selecting
an entity triggered `scrollIntoView` below the map; saves/reloads could trigger
it again. This made catalogue comparison and repeated edits cumbersome. The
management “New entity” control also only cleared the selection instead of
opening candidate creation.

## Current workflow

1. Filter the catalogue, choose 25/50/100 rows and use independent checkboxes
   for batch deletion. Unassigned entities remain available.
2. Click an entity name (or an entity in the optional catalogue map) to open
   the editor immediately. The underlying catalogue keeps its place.
3. Use **Name & categories**, **Location**, **Geometry**, **Source comparison**
   and **Audit & deletion** sections. **Previous entity / Next entity** browse
   only the loaded page; close the editor to change catalogue pages or filters.
4. Each editable section has an explicit Save action. Saving refreshes the
   selected entity/revision and audit via APIs without resetting other drafts.
   Concurrent changes still produce a conflict with an explicit discard/reload
   action; they are not silently overwritten.
5. Closing with **Close** or Escape, changing entity or navigating away asks
   before discarding unsaved drafts. Section changes retain drafts. Saves block
   entity navigation/closing until their request and refresh finish. Native
   modal dialogs contain keyboard focus, make the background inert and restore
   focus to the invoking control when it remains available.
6. **New Candidate** opens the existing manual proposal form/drawing flow;
   creation does not silently approve an entity. Review decisions and lifecycle
   transitions stay on `/geodata`, not in the catalogue editor.

## Geometry workspace

The geometry section has a focused Leaflet map using the same tile attribution,
status styling and Leaflet-Geoman integration as other admin maps. It starts
read-only. Enable **Edit geometry vertices**, or explicitly draw a replacement
point, trail or polygon. Drawing updates a draft only; **Save geometry** and
the change note submit the geometry through the API. The advanced GeoJSON
section retains Point/LineString/MultiLineString/Polygon/MultiPolygon support.
Type and coordinates must be compatible; server validation is authoritative.
Retired geometries remain read-only. Leaving a section stops interactive editing
but retains its draft. Container-size observation fixes map sizing in dialogs
and when reopening the collapsed catalogue map.

Location metadata retains manual precedence, read-only provider codes, queued
missing-field enrichment and worker-result polling. Location suggestions now
use the correct per-field administrative tree, independent of catalogue filters.
Maidenhead arrays remain automatically calculated and read-only.

## Deletion and service boundaries

Single deletion stays under **Audit & deletion**, restricted to global admins.
Its native confirmation dialog can appear above the editor. Bulk deletion stays
on the catalogue toolbar. Both preserve the existing durable job creation,
impact lookup, explicit confirmation, continuous polling, QSO cascade/award
warning, and automatic close on completion. Closing the progress dialog does
not cancel confirmed server jobs. No real entities are deleted by browser tests.

No new endpoints or contract changes are needed:

| Action | Existing API resource |
| --- | --- |
| Catalogue / focused resource / audit | `GET /v1/geodata/entities`, `GET /v1/geodata/entities/{id}`, `GET /v1/geodata/entities/{id}/audit` |
| Name and location | `PATCH /v1/geodata/entities/{id}` with `If-Match` |
| Categories / geometry | `PUT /v1/geodata/entities/{id}/categories`, `PUT /v1/geodata/entities/{id}/geometry` with `If-Match` |
| Missing location update | `POST /v1/geodata/entities/{id}/location-enrichment-requests` |
| Protected deletion | Existing entity-deletion-job create/read/confirm resources |

The browser never accesses PostgreSQL, NATS or object storage directly.
See the [navigation diagram](diagrams/entity-catalogue-editor.md).

## Regression checks

In `myota-admin-web`, run `npm test`, `npm run typecheck`, `npm run build`,
then `npx playwright install chromium` and `npm run test:e2e`.
Browser checks exercise actual Vue/Leaflet/Geoman against isolated API fixtures
with public tile requests blocked, not a live load-test dataset. They cover
draft guards, retained bulk selection/focus, partial saves, real vertex controls,
replacement drawing, retired restrictions, stacked deletion confirmations,
bulk completion, unchanged review decisions and a 390px mobile viewport.
CI retains desktop/mobile screenshots in `catalogue-browser-evidence`; these
are presentation evidence, not proof of production queue or database writes.
