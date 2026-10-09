# Phase 3 bounded preprocessing — implementation and exit review

**Review date:** 9 October 2026  
**Result:** Streaming/checkpoint and local recovery gates advanced; Phase 3 remains open.
**Environment:** Local service validation and disposable local PostGIS tests;
read-only K3s inspection only. No production imports, entity mutations, or
worker fault injection were run.

## Implemented and verified in this follow-up

- Uploaded object-store sources stream to worker-local scratch files rather
  than being copied into one Python `bytes` value. Candidate checkpoints use
  stable import ordinals, default to 100 records per batch, and evict committed
  row projections before advancing.
- GeoJSON FeatureCollections/arrays use `ijson`; KML yields placemarks, GPX
  yields waypoints and track segments, zipped Shapefile/ParkServe iterates
  records after archive/record preflight, and OSM PBF uses PyOsmium with a
  disk-backed sparse node-location index. Recovery metadata preserves snapshot
  policy and reopens the immutable source instead of retaining a decoded list.
- Decoder defaults cap a decoded feature at 16 MiB and 250,000 vertices.
  Shapefile additionally caps expanded archive size at 1 GiB, member count at
  100, and preflights record size before pyshp decodes geometry. Every import
  is limited to 5,000 features by default. Complete snapshots are preflighted
  against that limit and then replayed from source in one bounded window. These
  are enforceable input/work ceilings, not a universal worker-memory guarantee.
- Decoder fixtures cover the implemented upload formats, archive/record guard
  regressions, recovery format selection, and snapshot replay.
- An RSS subprocess test parsed 300,000 GeoJSON point features at **23.9 MB
  peak RSS** on macOS. Separate KML and GPX subprocesses each parsed 50,000
  small records at **20.5 MB** and **20.6 MB peak RSS**, respectively. Tests
  enforce a 96 MiB ceiling. These fixtures do not qualify a 1 GiB upload or
  worst-case geometry.
- In a disposable local PostGIS `*_tests` database, a worker was killed after
  its first committed 100-record checkpoint, reclaimed, and replayed to exactly
  250 candidates with 250 distinct ordinals and `PREPROCESSED` status. A
  separate two-process claim race produced exactly one lease owner. Fourteen
  relational concurrency tests, including promotion replay, also passed.
- An interrupted earlier local test run left rows in the disposable database.
  The test container and its anonymous data volume were removed after
  verification, erasing those fixtures. The live K3s cluster was not written.

## Remaining qualification gaps

- ElementTree may materialize a large individual XML text/element before the
  decoded-feature guard can reject it. Large OSM areas are also converted
  before the generic decoded-geometry check. The global upload-byte ceiling
  and feature-count limit bound total admitted work, but do not prove safe
  worst-case peak RSS for those parser objects.
- Peak RSS has not been measured for Shapefile, OSM PBF, complete-snapshot
  replay, remote WFS/ArcGIS adapter responses, or worst-case individual
  features.
- The full local suite collected 132 tests: 114 passed and 18 were skipped
  because isolated database/process services were not configured. The 16
  relevant database-backed worker/concurrency tests were run separately and
  passed.
- Failure injection directly covers forced death after a durable candidate
  checkpoint, database lease reclaim, replay, and competing preprocessing
  claims. It does not yet cover death while parsing before the first checkpoint,
  location enrichment, promotion between commit and JetStream ACK, or actual
  JetStream redelivery/ACK-pending behavior under competing consumers.
- Graceful shutdown/drain has code and unit coverage, but it has not been
  combined with each worker-stage crash/replay scenario in CI.

## Validation performed

- Ruff lint and format checks passed locally.
- Full local unit suite: **132 collected, 114 passed, 18 environment-dependent
  skips**. Streaming dependencies (`ijson`, `osmium`, and `pyshp`) were installed
  in a temporary Python environment, so parser/RSS tests ran rather than
  skipping.
- Database-backed recovery/concurrency subset: **16 passed** against the
  disposable local PostGIS database. The database container and volume were
  removed afterward; no test fixtures remain.
- No live writes, load tests, pod/service kills, or database/object-store
  changes were performed on K3s.
- Service implementation commit:
  [`b5b1acb`](https://github.com/myota-platform/myota-geodata-service/commit/b5b1acb1128f025049bbec30921663ed3949078b).
- Previous implementation/deployment evidence: service commit
  [`301bc28`](https://github.com/myota-platform/myota-geodata-service/commit/301bc28),
  deployment commit
  [`890197f`](https://github.com/myota-platform/myota-deploy/commit/890197f),
  and the associated passing build/chart/Fleet checks recorded in the prior
  evidence entry. These results predate this follow-up and do not verify its
  image until the new rollout completes.

## Remaining exit actions

1. Reject oversized XML/PBF objects before parser-object allocation, then
   measure Shapefile, PBF, snapshot, and remote adapter peak RSS on large
   representative fixtures.
2. Add isolated CI failure injection for worker death during parse,
   enrichment, candidate persistence, and promotion—graceful and forced—with
   lease reclaim and idempotent replay.
3. Run a disposable JetStream integration test for duplicate delivery,
   competing consumers, commit-before-ACK recovery, ACK-pending, and redelivery.
4. Append new image-build and read-only K3s deployment evidence after publishing
   and rolling out this follow-up.
5. Close Phase 3 only when the remaining memory and worker-stage gates have
   retained passing evidence.

See the [Phase 3 roadmap section](../horizontal-scaling-roadmap.md#phase-3--isolate-preprocessing-and-promotion-from-api-pods),
[active work index](../../work-in-progress/README.md), and
[prioritized backlog](../../to-do/prioritized-backlog.md).
