# Phase 3 bounded preprocessing — implementation and exit review

**Review date:** 9 October 2026  
**Result:** Phase 3 bounded preprocessing and recovery gates passed in an
isolated disposable K3s namespace. The application namespace, application
databases, object storage, and application workers were not used for test data
or fault injection.
**Environment:** Tests ran on `spainip.es` against a temporary PostGIS
`geodata_tests` database and a temporary JetStream broker in namespace
`myota-phase3-evidence-20261009`. Both test services were disposable and had no
persistent volumes. The final namespace teardown and normal-deployment health
check are recorded below.

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
- XML inputs now receive a fixed-chunk Expat preflight before ElementTree
  materialization. The preflight rejects DTDs, oversized feature subtrees, and
  excessive element counts before the feature tree is built. OSM area ring
  vertices are counted before PyOsmium's GeoJSON conversion. A real zipped
  Shapefile regression was fixed by extracting the three required sidecars to
  bounded disk scratch files; the current PyShp reader returned zero shapes
  when fed ZIP member streams directly.
- Preprocessing batches stop at either 100 features or 32 MiB of serialized
  source feature data, whichever comes first. A single feature may exceed the
  batch budget only if it remains within the separate 16 MiB decoded-feature
  ceiling; it is then isolated in its own batch. Complete snapshots use these
  same byte-bounded checkpoints after their feature-count preflight.
- Decoder fixtures cover the implemented upload formats, archive/record guard
  regressions, recovery format selection, and snapshot replay.
- Bounded RSS tests processed 300,000 features for each of GeoJSON, WFS
  response GeoJSON, and ArcGIS FeatureServer response GeoJSON; 50,000 records
  each for KML, GPX, zipped Shapefile, ParkServe's Shapefile adapter, and OSM
  PBF; and one valid 250,000-vertex GeoJSON LineString. A 1,000-feature
  complete-snapshot preprocessing run reopened its source twice and committed
  all candidates without retaining the decoded list. Final measured Linux
  process high-water RSS was 42.2 MB for the 300k JSON/XML families, 56.2 MB
  for zipped Shapefile/ParkServe and OSM PBF, 67.3 MB for the maximum-vertex
  GeoJSON feature, and 69.7 MB for complete-snapshot preprocessing. All stayed
  below the test ceiling of 96 MiB. A separate oversized-XML test verified
  rejection before ElementTree was called; an oversized OSM area test verified
  rejection before GeoJSON materialization.
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
- A final isolated PostGIS failure-injection run killed workers before the
  first parse checkpoint, after a committed 100-record checkpoint, during
  location-provider lookup before commit, and after promotion had materialized
  an entity in memory but before its durable checkpoint. Lease reclaim and
  replay completed each job without duplicate candidate ordinals or entities.
  A JetStream shutdown test set the stop signal while a delivery handler was
  blocked, released it, then verified acknowledgement and zero pending work.
- The service changes were merged in commit
  [`cf28693`](https://github.com/myota-platform/myota-geodata-service/commit/cf286931db526ad1c990ed9fa32a0db37a5ff98a),
  including the worker-fixture isolation fix from `504eeb3`. Python quality,
  load-harness, upload-session recovery, and relational-boundary CI checks all
  passed on the final pre-merge branch. The merged main-branch Python-quality
  job and image build/publish also passed ([run 37924979193](https://github.com/myota-platform/myota-geodata-service/actions/runs/37924979193),
  [run 37924977799](https://github.com/myota-platform/myota-geodata-service/actions/runs/37924977799)).
  The published image is `ghcr.io/myota-platform/myota-geodata-service:latest`
  at `sha256:04202907b6a0e872f14aa86d9a98c7932e1c10f5b6f29fa47672b7baef4ba275`.
- The disposable K3s namespace contained only temporary test database/broker
  workloads and no persistent volumes. The application namespace, durable
  databases, object storage, entities, imports, and application worker state
  were not modified.

## Qualification scope and limits

- Phase 3 gates are bounded to the enforced 16 MiB feature, 250,000-vertex,
  5,000-feature import, and 100-feature/32-MiB checkpoint settings. The results
  qualify these guards and representative datasets; they are not a claim that
  every possible 1 GiB input has the same RSS or throughput.
- WFS and ArcGIS measurements use representative response documents through
  the same streaming JSON decoder, not a live third-party endpoint. External
  source availability/latency is outside this parser qualification.
- The parser/recovery tests do not qualify a larger replica count or replace
  Phase 4/5 staged rollout and capacity tests.

## Validation performed

- Ruff lint and format checks passed on the exact files used for the run.
- Phase 3 focused suite: **33 tests passed, no skips**, using temporary PostGIS
  and JetStream services on the K3s host. It included the parser/RSS matrix,
  snapshot processing, database failure-injection/replay, and JetStream
  acknowledgement/drain cases. The expected simulated lost-ACK warning was
  emitted by its passing test.
- All temporary fixture rows were removed by test teardown. Namespace deletion
  removed the PostGIS and NATS pods/services and their ephemeral storage; the
  temporary checkout, virtual environment, and port-forward processes were also
  removed. No application database, import, entity, or object-storage data was
  changed by qualification.
- After cleanup, the existing Fleet application workloads remained ready and
  the public gateway health endpoint was checked. The deployed application was
  not stopped or restarted during fault injection.
- Service merge commit:
  [`cf28693`](https://github.com/myota-platform/myota-geodata-service/commit/cf286931db526ad1c990ed9fa32a0db37a5ff98a). Python quality, load
  harness, upload-session recovery, and relational-boundary CI passed. The
  [main-branch image build](https://github.com/myota-platform/myota-geodata-service/actions/runs/37924977799)
  published digest
  `sha256:04202907b6a0e872f14aa86d9a98c7932e1c10f5b6f29fa47672b7baef4ba275`.
- Deployment commit
  [`54ac990`](https://github.com/myota-platform/myota-deploy/commit/54ac990cc156ef85d841ab089284b8c82a1c7685)
  pins that digest for Fleet. Helm rendering and deployment-repository quality
  passed ([chart run 37925422702](https://github.com/myota-platform/myota-deploy/actions/runs/37925422702),
  [quality run 37925423764](https://github.com/myota-platform/myota-deploy/actions/runs/37925423764)).
  Fleet observed commit `54ac990cc156ef85d841ab089284b8c82a1c7685` and reported
  54/54 resources ready. Both `myota-geodata` and
  `myota-geodata-import-processing` completed rollout and use the pinned digest;
  `https://api.myota.top/healthz` returned `{"status":"ok","service":"gateway"}`.
  The disposable test namespace was absent after teardown. No application data
  or object-storage content was removed.
- Previous implementation/deployment evidence: service commit
  [`301bc28`](https://github.com/myota-platform/myota-geodata-service/commit/301bc28),
  deployment commit
  [`890197f`](https://github.com/myota-platform/myota-deploy/commit/890197f),
  and the associated passing build/chart/Fleet checks recorded in the prior
  evidence entry. Those older results are historical and are superseded for
  this image by deployment commit 5afb183 and the current read-only checks
  above.

## Remaining exit actions

1. [x] Publish the merged service image, update the chart's pinned digest, and
   verify the API and import-processing worker rollouts plus public health.
2. Keep Phase 4 infrastructure review and Phase 5 staged rollout qualification
   open; Phase 3's bounded correctness results do not qualify larger replica
   counts or production capacity.

See the [Phase 3 roadmap section](../horizontal-scaling-roadmap.md#phase-3--isolate-preprocessing-and-promotion-from-api-pods),
[active work index](../../work-in-progress/README.md), and
[prioritized backlog](../../to-do/prioritized-backlog.md).
