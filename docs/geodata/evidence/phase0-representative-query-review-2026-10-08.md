# Phase 0 representative catalogue review — 8 October 2026

This sanitized artifact records the guarded production PostGIS plan capture,
bounded read-only API profile, and the contemporaneous object-storage and
worker signals. The exact-host allowlist was `myota-geo-postgis`; statements
were read-only with a 5-second timeout. No credentials, entity identifiers,
or source payloads are retained here.

## Fixture cardinality and status

Four imports carried 2,500 synthetic point features each (10,000 input
features total) into the permanent `SCALE_TEST_FIXTURE` category. After the
requested sample change, the three remaining imports each promoted 5% (2.5%
Candidate, 2.5% Approved) and rejected the other 95% from staging. Because
2.5% of 2,500 is fractional, their counts alternate 62/63.

The first import had already been queued to approve all 2,500 records before
the change; Approved entities cannot be downgraded. It was retained as a
documented exception. The final permanent catalogue contains **2,875
synthetic entities: 2,688 Approved and 187 Candidate**. All four import runs
were finalized, and rejected/processed staged records were discarded. These
fixtures are not real parks and are not assigned to a programme.

## Guarded `EXPLAIN (ANALYZE, BUFFERS)`

Captured at `2026-10-08T13:43:03Z` on PostgreSQL 16.4, after fixture processing.
The map/count window was `(-5.99, 37.37, -5.90, 37.43)` and page limit was 100.
The planner estimated 2,833 rows for `geodata_entity`; the actual catalogue
contained 2,875.

| Query | Plan/access path | Estimated vs actual rows | Execution time | Shared buffers / spills |
| --- | --- | ---: | ---: | --- |
| Map bounds + `ST_Intersects`, ordered and limited | Bitmap Heap Scan using `geodata_entity_geom_idx`; top-N heapsort in memory | Spatial scan 9 vs 278; 100 rows returned by the limit | 2.053 ms | 62 hits, 0 reads; 0 temp I/O |
| Catalogue count in map window | Bitmap Heap Scan using `geodata_entity_geom_idx`, then aggregate | 159 vs 278 matches | 0.098 ms | 27 hits, 0 reads; 0 temp I/O |
| Catalogue page ordered by `lower(name), id` | Index Scan using `geodata_entity_catalogue_sort_idx` | 2,880 estimated table rows; 100 returned | 3.022 ms | 332 hits, 0 reads; 0 temp I/O |

The spatial GiST index was used for both the map and count plans. The map
selectivity estimate (9) was about 31× below the 278 rows found by the index
scan; the count estimate was about 1.7× low. The table cardinality estimate
was close to actual. The page query used the dedicated catalogue sort index
and avoided an explicit sort. All three queries had zero disk reads and no
sort/hash spill in this warm-cache capture. The map estimate mismatch merits
follow-up with broader data distributions and planner-statistics review; it is
not evidence that the GiST index is missing.

## Bounded read-only API profile

The existing production-guarded k6 script ran against `https://api.myota.top`
for 60 seconds at 2 VUs. It sampled 10 entities in memory and created no
application data.

| Measurement | Result |
| --- | ---: |
| Iterations | 60 |
| HTTP requests | 241 |
| Failed requests | 0 (0%) |
| Checks | 242 / 242 passed |
| Overall HTTP p95 | 28.75 ms |
| Maximum HTTP request | 62.6 ms |
| Tested operations | Health, paged catalogue, bbox/map catalogue, entity detail |

The Prometheus five-minute HTTP histogram showed geodata catalogue route p95
of about 24.2 ms, entity-detail p95 about 4.8 ms, and health p95 about 4.8 ms.
These are low-concurrency, short-window observations, not a saturation or
capacity guarantee.

## Object storage and worker correlation

After the fixture imports and read profile, the live scrape reported:

- SeaweedFS `myota-geodata-imports` activity over the recent 15-minute window:
  approximately 6 successful GETs and 3 successful PUTs; the corresponding
  observed histogram p95s were about 10.9 ms (GET) and 25.0 ms (PUT). The
  sample is small and counter windows are extrapolated by Prometheus, so it is
  descriptive rather than a storage capacity estimate.
- The geodata PostGIS histogram reported about 153 catalogue-query
  observations in the recent five-minute window with p95 about 23.9 ms.
- Import queue depth and active processing runs were 0 after finalization.
  JetStream pending, ack-pending, redeliveries, and oldest-message age were 0
  for the observed durable consumers. No sustained queue backlog was visible.

No object-storage or worker backlog bottleneck was observed at this bounded
workload. The read profile's API p95 and the SQL timings were similar in scale;
the single EXPLAIN capture had a warm cache and should not be extrapolated to
cold storage or high concurrency.

## Conclusion and limits

The Phase 0 representative-query review is complete for the **2,875-entity
catalogue actually created under the user's 5% promotion instruction**, with
the first full-approval batch explicitly called out. Spatial and catalogue
indexes were selected; the measured plans and 2-VU read profile were healthy
for this dataset. The map cardinality estimate is materially low and should be
investigated before using planner estimates for larger regional catalogues.

This evidence does not establish capacity at 10,000 promoted entities, tens of
thousands of users, millions of QSOs, cold-cache conditions, or multiple
geodata replicas. It closes the current Phase 0 evidence gate only at the
measured fixture size. Stronger cardinality and concurrency goals remain later
scale work; do not describe this result as broad production qualification.

Related records: [Phase 0 production evidence](phase0-production-evidence-2026-10-08.md),
[pre-fixture query plan](phase0-current-catalogue-plan-2026-10-08.md),
[load/query runbook](load-test-and-query-evidence.md),
[scaling roadmap](../horizontal-scaling-roadmap.md).
