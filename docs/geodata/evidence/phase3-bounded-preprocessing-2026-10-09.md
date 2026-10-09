# Phase 3 bounded preprocessing — implementation and exit review

**Review date:** 9 October 2026  
**Result:** Partial implementation; Phase 3 remains open.  
**Environment:** local service unit tests only. No production jobs were
terminated, restarted, or modified.

## Implemented in this delivery

- Large uploaded source objects are copied from SeaweedFS to a worker-local
  temporary file using bounded I/O instead of `get_object(...).read()` into one
  Python `bytes` object. The existing upload size ceiling is checked before
  the download.
- GeoJSON FeatureCollections and top-level feature arrays are decoded with
  `ijson` incrementally. Each candidate uses its stable source ordinal; the
  worker queries only the current ordinal range, commits the configured batch,
  and evicts committed candidate projections before continuing. The default
  batch is 100 and can be tuned with `MYOTA_IMPORT_BATCH_SIZE` in Compose and
  Helm values (`geodataImportProcessing.batchSize`).
- Replayed batches re-use their existing staged IDs and preserve validation,
  processing, and audit-related state. Run feature-count progress is persisted
  at each checkpoint. `ijson` is installed in the service image requirements.
- Import parser and bounded-window unit coverage was added to the geodata
  service. The test suite verifies ordinal-window behavior, batch checkpoints,
  candidate preservation, and iterator compatibility.

## Limits that keep the gate open

The bounded path is not yet universal:

- KML, GPX, Shapefile, and remaining accepted binary/adapter formats still use
  their existing whole-document decoder or parser-adapter path.
- `completeSnapshot` currently materializes the full feature set because its
  disappearance semantics compare the entire snapshot. It must be redesigned
  with an incremental manifest before it can use batch checkpoints safely.
- If `ijson` is absent, GeoJSON keeps a compatibility fallback that reads the
  complete source. The published production image installs it, but the local
  environment had no network access to install/test the dependency branch.
- Result/manifest bookkeeping retains per-record identifiers and hashes, so
  metadata is still proportional to import feature count even when geometry
  payloads are windowed.
- No peak-RSS measurement against a representative large fixture was captured.
- No isolated worker process/pod kill test was run during parsing, enrichment,
  candidate persistence, or promotion. Forced termination, lease reclaim,
  duplicate delivery under two workers, and concurrent worker claims remain
  unverified.

## Validation performed

- Ruff lint and format checks passed locally.
- Full geodata unit suite: 121 tests passed, 17 environment-dependent tests
  skipped.
- Focused worker/import suite: 32 tests passed; one streaming-dependency test
  skipped because `ijson` could not be installed on this host.
- GitHub Actions `Python quality` passed on service commit `301bc28`; its
  PostGIS-backed two-API regression job applied every service migration,
  installed `requirements.txt` (including `ijson`), and passed all 121 tests
  without skips. This verifies the actual streaming parser test path, but is
  not a forced worker-termination or peak-RSS test.
- The local test environment lacked `ijson`; an attempted dependency install
  could not reach PyPI. The test that exercises the real streaming parser is
  therefore present but locally skipped. It must pass in CI after the image
  workflow installs `requirements.txt` before considering this parser path
  verified.
- Service commit: [`301bc28`](https://github.com/myota-platform/myota-geodata-service/commit/301bc28).
- Deployment configuration commit: [`890197f`](https://github.com/myota-platform/myota-deploy/commit/890197f).
- [Python quality and PostGIS-backed CI run](https://github.com/myota-platform/myota-geodata-service/actions/runs/37913194172) passed.
- [Helm chart validation](https://github.com/myota-platform/myota-deploy/actions/runs/37913227459) passed. The geodata image publication and Fleet rollout are not yet verified in this record.
- No isolated NATS worker-recovery test or Helm/Fleet rollout is claimed by
  this evidence record.

## Remaining exit actions

1. Make all advertised formats and complete-snapshot processing bounded or
   define an enforceable safe fallback per format.
2. Add a representative large-fixture RSS test and retain its result.
3. Add isolated CI failure injection for graceful and forced worker termination
   at each lifecycle boundary, with lease reclaim and idempotent replay.
4. Exercise duplicate deliveries and concurrent worker claims using at least
   two workers against an isolated PostGIS database and JetStream instance.
5. Close the Phase 3 checklist only after the above evidence passes.

See the [Phase 3 roadmap section](../horizontal-scaling-roadmap.md#phase-3--isolate-preprocessing-and-promotion-from-api-pods),
[active work index](../../work-in-progress/README.md), and
[prioritized backlog](../../to-do/prioritized-backlog.md).
