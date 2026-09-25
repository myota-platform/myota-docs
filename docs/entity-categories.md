# Shared entity categories and programme assignments

Entity categories are shared master data. MyOTA does not impose a universal list: administrators can define `MUNICIPAL_PARK`, `NATURE_RESERVE`, `TRAIL`, or any other programme-independent category, then assign the same category to one or more programmes.

The programme service manages the shared catalogue through:

- `GET /v1/entity-types`
- `POST /v1/entity-types`

Each category has a stable uppercase `code`, display `label`, one or more supported geometry kinds (`POINT`, `LINESTRING`, `MULTILINESTRING`, `POLYGON`, or `MULTIPOLYGON`), optional description, and active flag. Codes cannot be renamed because imports, awards, and historical activity may reference them. Inactive categories remain available for historical display but are not offered for new assignments.

The administration Master data page edits shared category definitions. Programme Management provides the assignment function:

- `GET /v1/programmes/{slug}/entity-types`
- `POST /v1/programmes/{slug}/entity-types/assign`
- `POST /v1/programmes/{slug}/entity-types/unassign`

The same category can therefore be assigned to multiple programmes without duplicating or redefining it. The Geodata Review page loads the selected programme’s assigned catalogue and allows an authorized reviewer to change an entity’s category. The geodata service records the previous and new codes, editor, note, and timestamp in the entity audit history.

Changing a category assignment does not change geometry. Changing a shared category definition affects every programme to which it is assigned, so definition changes should be reviewed before publication or use; the category’s geometry types are catalogue hints and import validation contracts. Existing single `geometry` values are read as a one-item `geometryTypes` list for compatibility.
