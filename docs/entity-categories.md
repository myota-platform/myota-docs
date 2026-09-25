# Programme-owned entity categories

Entity categories are configured by each programme. MyOTA does not impose a universal list: one programme may define `MUNICIPAL_PARK`, another may define `NATURE_RESERVE`, and a trail-oriented programme may define `TRAIL`.

The programme service manages category records through:

- `GET /v1/programmes/{slug}/entity-types`
- `POST /v1/programmes/{slug}/entity-types`

Each category has a stable uppercase `code`, display `label`, supported geometry kind (`POINT`, `LINESTRING`, `POLYGON`, or `MULTIPOLYGON`), optional description, and active flag. Codes cannot be renamed because imports, awards, and historical activity may reference them. Inactive categories remain available for historical display but are not offered for new assignments.

The administration Programme Editor provides a form-driven category manager while retaining an advanced JSON view for programme-specific extensions. The Geodata Review page loads the selected programme’s active catalogue and allows an authorized reviewer to change an entity’s category. The geodata service records the previous and new codes, editor, note, and timestamp in the entity audit history.

Changing a category does not change geometry. Geometry compatibility is a programme administration responsibility and should be reviewed before approval or activation; the category’s geometry kind is a catalogue hint and import validation contract.
