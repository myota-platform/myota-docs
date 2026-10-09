# Phase 3 bounded preprocessing — implementation and exit review

**Review date:** 9 October 2026  
**Result:** Streaming/checkpoint and local recovery gates advanced; Phase 3 remains open.
**Environment:** Local service validation and disposable local PostGIS tests.
JetStream delivery tests run in GitHub CI against a disposable broker and the
isolated *_tests database. The final image was rolled out through Fleet/Helm
to K3s and verified read-only. No production imports, entity mutations, or
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
- Preprocessing batches stop at either 100 features or 32 MiB of serialized
  source feature data, whichever comes first. A single feature may exceed the
  batch budget only if it remains within the separate 16 MiB decoded-feature
  ceiling; it is then isolated in its own batch. Complete snapshots use these
  same byte-bounded checkpoints after their feature-count preflight.
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
- The worker's JetStream delivery path is exercised in CI with a disposable
  in-memory stream: ACK-pending is observed while a handler is blocked and
  returns to zero after acknowledgement; a simulated lost ACK after the
  processed-event commit redelivers without invoking the handler twice; and
  two pull consumers drain eight events with exactly-once handler effects.
  These broker tests qualify delivery/idempotency semantics, not forced
  process death at every business-worker stage.
- The final GitHub Actions rerun for service commit
  [`2de096f`](https://github.com/myota-platform/myota-geodata-service/commit/2de096f90d3a68092e69c0456246c6a1e0e67d59)
  passed all **137 tests**
  with no skips, including disposable JetStream and PostGIS recovery tests. The
  CI RSS subprocesses processed 300,000 GeoJSON features and 50,000 KML and GPX
  records; peak process RSS remained about **61.5 MB**, below the 96 MiB test
  ceiling. The [final CI run, attempt 2](https://github.com/myota-platform/myota-geodata-service/actions/runs/37920214726)
  also passed the isolated SeaweedFS upload-recovery and load-harness jobs, and
  published image digest `sha256:97cf732f9f14f8e5ff76b665b9a893f78033b7e954db80ad8ca2fe3d5ff74780`.
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
- The full local suite collected 137 tests: 116 passed and 21 were skipped
  because isolated database/process services were not configured. The 16
  relevant database-backed worker/concurrency tests were run separately and
  passed.
- Failure injection directly covers forced death after a durable candidate
  checkpoint, database lease reclaim, replay, and competing preprocessing
  claims. It does not yet cover death while parsing before the first checkpoint,
  location enrichment, or promotion-stage process death. JetStream
  commit-before-ACK, redelivery, ACK-pending and competing-consumer semantics
  have isolated integration coverage but do not substitute for those business
  worker-stage process-death tests.
- Graceful shutdown/drain has code and unit coverage, but it has not been
  combined with each worker-stage crash/replay scenario in CI.

## Validation performed

- Ruff lint and format checks passed locally.
- Full local unit suite: **137 collected, 116 passed, 21 environment-dependent
  skips**. Streaming dependencies (`ijson`, `osmium`, and `pyshp`) were installed
  in a temporary Python environment, so parser/RSS tests ran rather than
  skipping.
- Database-backed recovery/concurrency subset: **16 passed** against the
  disposable local PostGIS database. The database container and volume were
  removed afterward; no test fixtures remain.
- No live imports, entity/test-fixture writes, load tests, pod/service kills,
  or manual database/object-store changes were performed on K3s. The only
  live change was the requested Helm/Fleet image rollout.
- Service implementation commit:
  [`2de096f`](https://github.com/myota-platform/myota-geodata-service/commit/2de096f90d3a68092e69c0456246c6a1e0e67d59); its final CI rerun passed.
- Deployment commit
  [`5afb183`](https://github.com/myota-platform/myota-deploy/commit/5afb18379a153ef04ad7cfe310e5efe81d4b102d)
  pins the published image for Fleet rollout. Both myota-geodata and
  myota-geodata-import-processing pods report the expected OCI image
  sha256:97cf732f9f14f8e5ff76b665b9a893f78033b7e954db80ad8ca2fe3d5ff74780;
  the public gateway /healthz endpoint returned
  {"status":"ok","service":"gateway"}. Helm chart validation and
  deployment-repository quality checks passed (runs
  [37920788467](https://github.com/myota-platform/myota-deploy/actions/runs/37920788467)
  and [37920789435](https://github.com/myota-platform/myota-deploy/actions/runs/37920789435)).
- Previous implementation/deployment evidence: service commit
  [`301bc28`](https://github.com/myota-platform/myota-geodata-service/commit/301bc28),
  deployment commit
  [`890197f`](https://github.com/myota-platform/myota-deploy/commit/890197f),
  and the associated passing build/chart/Fleet checks recorded in the prior
  evidence entry. Those older results are historical and are superseded for
  this image by deployment commit 5afb183 and the current read-only checks
  above.

## Remaining exit actions

1. Reject oversized XML/PBF objects before parser-object allocation, then
   measure Shapefile, PBF, snapshot, and remote adapter peak RSS on large
   representative fixtures.
2. Add isolated CI failure injection for worker death during parse,
   enrichment, candidate persistence, and promotion—graceful and forced—with
   lease reclaim and idempotent replay. Candidate-checkpoint death/replay and
   database lease competition are covered; the other worker stages remain open.
3. JetStream delivery is qualified for ACK-pending release,
   commit-before-ACK recovery, redelivery and competing consumers in the linked
   CI run. Forced business-handler process death remains a separate open gate.
4. [x] Publish and roll out the final image; verify the API and worker image
   IDs plus public health. See deployment commit and CI links above.
5. Close Phase 3 only when the remaining memory and worker-stage gates have
   retained passing evidence.

See the [Phase 3 roadmap section](../horizontal-scaling-roadmap.md#phase-3--isolate-preprocessing-and-promotion-from-api-pods),
[active work index](../../work-in-progress/README.md), and
[prioritized backlog](../../to-do/prioritized-backlog.md).
