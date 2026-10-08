# Pre-fixture catalogue query-plan snapshot — 8 October 2026

Captured before the permanent synthetic fixture import from the live
provisional-production `myota-geo-postgis` host using
the published guarded plan tool inside the geodata pod. The connection was
read-only, each statement had a 5-second timeout, the spatial window was
Seville-area `(-5.99, 37.37, -5.90, 37.43)`, and the page limit was 100. No
database changes were made.

## Catalogue size and indexes

- PostgreSQL: 16.4.
- `pg_class.reltuples` planner estimate for `geodata_entity`: 5 rows; the
  ordered catalogue-page scan returned 0 rows.
- GiST indexes exist on both `geom` and `(geom)::geography`.
- Map query returned 0 rows in this window.

## Plan summary

| Query | Estimated rows | Actual rows | Access path | Buffers | Execution |
| --- | ---: | ---: | --- | --- | ---: |
| Map bounds + `ST_Intersects`, ordered, limit 100 | 5 | 0 | Sequential scan, then sort | 5 shared hits; 0 reads | 0.060 ms |
| Catalogue count for the map window | 1 aggregate; scan estimate 5 | 1 aggregate; 0 matching rows | Sequential scan | 2 shared hits; 0 reads | 0.012 ms |
| Catalogue page, ordered by `lower(name), id`, limit 100 | 5 | 0 | Sequential scan, then sort | 5 shared hits; 0 reads | 0.017 ms |

For the map query, PostgreSQL applied the GiST-compatible `geom && envelope`
predicate and exact `ST_Intersects` as a filter on the sequential scan; it did
not select the GiST index. With only five estimated table rows and no matching
entities, a sequential scan is expected and is not evidence of an index defect.
The timings are below one tenth of a millisecond after planning, with no disk
reads, but are not representative of catalogue-scale execution.

## Interpretation and gate

This snapshot is useful as a post-redeployment smoke baseline and confirms the
live query shape, current indexes, planner estimate, and buffer/timing fields.
It cannot answer whether the spatial or catalogue indexes are chosen at scale,
how estimates behave with a realistic distribution, or how latency changes as
the result set grows. This empty/near-empty catalogue does not meet the
scale-level gate.

The permanent synthetic Sevilla input set is 10,000 records in four imports
of 2,500. The requested sample promotes 5% per import, but the first batch
was already queued for full approval before that change and is retained as an
exception; see the [production evidence](geodata-phase0-production-evidence-2026-10-08.md#provisioning-exception-and-current-progress).
This pre-fixture snapshot does not represent the current catalogue after
provisioning. Capture a new plan at the resulting 2,875-entity cardinality and
review actual versus estimated rows, index conditions, sort work, buffers,
and execution time. Do not check off Phase 0 based on this low-cardinality
plan.

Related evidence: [large-upload and live storage/worker correlation](geodata-phase0-storage-correlation-2026-10-08.md),
[Phase 0 production record](geodata-phase0-production-evidence-2026-10-08.md),
[plan-capture runbook](geodata-load-test-and-query-evidence.md).
