# Maidenhead locator fields for geodata entities

Each entity exposes two automatically calculated, read-only arrays:

- `maidenheadGridSquares4`: unique four-character Maidenhead grid squares.
- `maidenheadLocators6`: unique six-character Maidenhead locators.

The arrays are sorted lexically for stable API responses. A point has one
canonical cell at each precision. For `LineString`, `MultiLineString`,
`Polygon`, and `MultiPolygon`, the arrays contain every cell whose rectangular
area intersects the geometry. Consequently, a line or polygon touching a grid
boundary can appear in the cells on both sides of that boundary. This
intersection rule is intentional: the fields represent geometry coverage, not
just the locator of a centroid. Empty geometry produces empty arrays.

The six-character form uses the standard lowercase subsquare letters (`a`–`x`)
after the two field letters and two square digits. Four-character cells are
2° longitude by 1° latitude; six-character cells subdivide these into
5′ longitude by 2.5′ latitude cells.

## Persistence and API behavior

The geodata service owns the calculation and storage. Migration
`019_maidenhead_locators.sql` adds `text[]` columns to `geodata_entity`,
backfills existing rows, and installs a `BEFORE INSERT OR UPDATE OF geom`
trigger. This database trigger is the persistence safeguard for imports,
geometry edits, and other geometry writers. API responses project the stored
columns as `maidenheadGridSquares4` and `maidenheadLocators6`; clients must not
submit or edit them. The OpenAPI contract marks both fields `readOnly`.

The admin Entity Catalogue displays both arrays as read-only values in the
entity list and selected-entity details. They are not location metadata and
cannot be manually overridden.

The canonical migration is maintained in
`myota-geodata-service/migrations/`. Its complete ordered migration set must
be synchronized byte-for-byte to
`myota-platform/db/migrations/geo/` and
`myota-deploy/db/migrations/geo/`, which are consumed by integration and
deployment. Apply the geodata migration before rolling out service instances
that project the new columns. The Helm migration runner discovers numbered
`geo/NNN_*.sql` files in lexical order, so each new migration is included in
the migration image automatically.

## Verification cases

The service test suite checks the known six-character Paris locator `JN18du`,
a single-cell point near Seville (`IM77` / `IM77aj`), a small polygon contained
in one cell, a polygon centered on the intersection of four four-character
grid squares (and four corresponding six-character cells), and a line crossing
two six-character locators. These cases cover both single-cell and
multi-cell geometry, including boundary contact. Database migration checks
should additionally verify that the trigger updates both arrays after a
geometry edit and that existing rows are backfilled.

The calculated values are geometric coverage only. They do not assert that an
entity is accessible, eligible under a programme's rules, or a valid radio
activation location.
