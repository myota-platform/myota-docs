# Phase 0 storage and worker correlation — 8 October 2026

This is a sanitized record of the bounded large-upload run performed after the
SeaweedFS scrape correction reached the live provisional-production K3s
deployment. It supplements, but does not replace, the representative-cardinality
PostGIS plan review required to close Phase 0.

## Workload and result

The existing `large-upload` k6 profile ran from `spainip.es` against
`https://api.myota.top`, with one virtual user and one iteration. It uploaded
2,500 synthetic features (3,606,069 bytes) using the resumable-upload API. The
run identifier was `lt-20261008-phase0-storage-correlation`.

- Checks: 3/3 passed (session creation, part checksum/storage, upload accepted).
- HTTP requests: 7; failed requests: 0.
- k6 p95 bulk-transfer request time: 76.43 ms.
- k6 p95 control-request time: 235.77 ms; its configured threshold was 2 s.
- Overall HTTP p95: 1.24 s; this includes control and bulk-transfer classes.
- The harness waited for processing to finish and its automatic cleanup
  reported `cleaned=true`, deleting one import, one object, and one upload
  session. A follow-up read showed zero geodata import queue depth, zero
  processing runs, and zero pending/ack-pending messages for the sampled
  `MYOTA_EVENTS` consumers.

No credentials, access tokens, feature contents, public callsigns, or
user-owned records are included here.

## Storage and service observations

The live SeaweedFS endpoint at private port 9324 exposed genuine S3 request
counters and request-duration histograms under
`exported_job="seaweedfs"`. In the post-run sample, the
`myota-geodata-imports` bucket showed successful GET, POST, PUT, and DELETE
traffic (HTTP 200/204); the measured PUT duration sum was about 34 ms for one
request. The in-flight upload byte gauge had returned to zero after cleanup.
The one-request PUT sample is descriptive only: it is not a stable percentile
or a storage-capacity estimate. A separate LIST/403 counter was also present;
because no before/after counter delta was retained for that series, this report
does not attribute it to the workload.

The API-side request measurements and SeaweedFS counters now demonstrate that
the expected upload/object lifecycle is visible independently from API
latency. The sample did not reveal an upload or queue bottleneck at this
bounded workload. It cannot establish throughput at higher concurrency,
object-store saturation, or worker capacity under sustained backlog.

The same Prometheus sample reports `deployment_environment="production"` and
stable gateway route templates for the session, part, completion, and cleanup
endpoints (with `{uploadId}`, `{partNumber}`, and `{testRunId}` placeholders).
Both gateway and geodata service emit corresponding route series; no concrete
UUIDs or load-test run IDs appear in the route labels.

## PostGIS plans and remaining work

This upload profile did not exercise the map bounding-box query. Existing
`EXPLAIN (ANALYZE, BUFFERS)` artifacts still cover only a 25-row catalogue and
are not scale evidence. A permanent 10,000-point synthetic Sevilla fixture
provisioner is ready in
[`myota-geodata-service/loadtests/provision_scale_fixtures.py`](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/provision_scale_fixtures.py).
It is source-tagged, unassigned to any programme, permanent, and intentionally
has no cleanup operation. Its size must be confirmed before execution. After
approval, capture map, catalogue-count, and catalogue-page plans with actual
and estimated rows, GiST/index conditions, buffers, and execution timing; then
correlate with a bounded map read and current worker/broker metrics.

**Phase 0 remains open.** This artifact closes only the live SeaweedFS
endpoint and bounded upload-correlation subtask. It does not close the
representative PostGIS query-plan gate.

Related records: [Phase 0 evidence and remaining gate](geodata-phase0-production-evidence-2026-10-08.md),
[load-test/query evidence runbook](geodata-load-test-and-query-evidence.md),
[scaling roadmap](geodata-horizontal-scaling-roadmap.md).
