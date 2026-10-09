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

The same category can therefore be assigned to multiple programmes without duplicating or redefining it. The Geodata Review page loads the complete shared catalogue, not only the categories assigned to the selected programme. An authorized reviewer can change an entity’s category even when the entity has no programme assignment; programme assignment remains a separate eligibility decision. The geodata service records the previous and new codes, editor, note, and timestamp in the entity audit history.

An entity may have more than one category. The geodata API accepts `entityTypes` (or the compatibility alias `entityTypeCodes`) as an ordered list of stable category codes. The first code is the primary value and is also exposed as the legacy singular `entityType`; the remaining codes are additional classifications. New candidate proposals and imports must select at least one category, and the administration UI uses a database-backed multi-select rather than a free-text field. Category filters match any assigned category.

The relational source of truth for these assignments is `geodata_entity_category(entity_id, category_id, category_code, is_primary)`. The JSON snapshot may contain `entityTypes` for compatibility and export, but it is not the authoritative multi-category store. The geodata service migration `008_entity_category_assignments.sql` is canonical; synchronized copies in `myota-platform` and `myota-deploy` must be updated in the same change and must not be edited independently.

Geodata Review also allows an authorized reviewer to correct an entity’s display name. Name edits are separate from source provenance and are recorded in the same audit history with the previous name, new name, editor, note, and timestamp.

Changing a category assignment does not change geometry. Changing a shared category definition affects every programme to which it is assigned, so definition changes should be reviewed before publication or use; the category’s geometry types are catalogue hints and import validation contracts. Existing single `geometry` values are read as a one-item `geometryTypes` list for compatibility.
