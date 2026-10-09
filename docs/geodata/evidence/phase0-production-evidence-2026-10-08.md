# Geodata Phase 0 provisional-production evidence — 8 October 2026

This sanitized record covers bounded tests against the user-designated live
K3s deployment on `spainip.es` (`https://api.myota.top`). Tests ran manually
from the server using the dedicated test account. Passwords, tokens, database
URLs, source payloads, and entity identifiers are intentionally omitted. Every
write profile used the harness's production opt-in and exact-host allowlist;
all tagged fixtures were successfully removed before the next profile.

## Read-only API baseline

The corrected read-only k6 baseline ran for 60 seconds at 2 VUs on 8 October
2026. Since the production catalogue is empty, it measured health and paged
catalogue reads and explicitly skipped map and entity-detail requests.

| Result | Value |
| --- | ---: |
| HTTP requests | 161 |
| Iterations | 80 |
| HTTP failures | 0 / 161 (0%) |
| Checks | 162 / 162 passed |
| Overall HTTP p95 | 7.01 ms |
| Created application data | None |

This qualifies only health/catalogue behavior at empty-catalogue cardinality;
it is not evidence for map/detail latency or realistic catalogue load. After
the user authorized temporary representative records, a second read-only run
sampled five tagged approved geometries in the Sevilla test extent:

| Result | Value |
| --- | ---: |
| Read profile | 2 VUs / 30 seconds |
| HTTP requests | 121 |
| Iterations | 30 |
| HTTP failures | 0 / 121 (0%) |
| Checks | 122 / 122 passed, including map and entity detail |
| Overall HTTP p95 | 9.53 ms |
| Temporary source | Promotion run `lt-1791462249228-237856476` |
| Cleanup | 5 entities, 1 import and 1 object removed |

Those records were synthetic temporary test geometries, not real park data;
none remain in the catalogue. Five records exercise the routes but do not
represent production catalogue cardinality.

## Write-profile results

All runs completed with zero failed HTTP requests, passing checks, and
successful exact-tag cleanup. Control-plane p95 is reported separately from
bulk upload timing where the harness provides that dimension.

| Profile / run ID | Bounded settings and work | Result | Cleanup |
| --- | --- | --- | --- |
| `large-upload` — `lt-1791461590688-801694184` | 1 VU; one 3,516,069-byte resumable GeoJSON upload | 7 requests; 0 failed; all 3 checks passed; bulk-transfer p95 114.74 ms; control p95 65.62 ms | 1 import, 1 upload session, 1 object removed |
| `simultaneous-edits` — `lt-1791461617004-887825197` | 5 VUs for 30 s; one shared test-owned entity | 162 requests; 0 failed; 151/151 checks passed; control p95 38.49 ms; 150 edit iterations | 1 import, 1 entity, 1 object removed |
| `preprocessing` — `lt-1791461662016-281368121` | 1 VU for 30 s; 5 imports × 100 features | 8 requests; 0 failed; 5/5 import checks passed; control p95 55.90 ms | 5 imports and 5 objects removed |
| `promotion` — `lt-1791461719666-62572792` | 1 VU for 60 s; 1 import × 25 features, validated and promoted | 9 requests; 0 failed; 4/4 checks passed; control p95 88.25 ms | 1 import, 25 entities, 1 object removed |
| `queue-backlog` — `lt-1791461968069-363347912` | 1 import/s for 30 s; 30 imports × 50 features | 33 requests; 0 failed; 30/30 checks passed; control p95 55.59 ms; one scheduled iteration dropped by the arrival-rate executor | 30 imports and 30 objects removed |

A shorter 10-second queue-backlog run also passed (10 imports accepted, zero
request failures, all 10 checks passed, control p95 61.80 ms) and cleaned all
10 imports and objects. The 30-second run is the retained backlog result.

These are low-to-moderate bounded production observations at the current
single-geodata-API deployment, not a saturation test or a guarantee for a
future replica count. The upload timing is client-observed end-to-end transfer
time; it does not isolate SeaweedFS processing time.

## PostGIS query-plan capture

The guarded query-plan tool ran inside the geodata pod with a 5-second
statement timeout and read-only session while the `promotion` run's 25 tagged
entities were present. Capture time was `2026-10-08T12:16:11.894233Z`;
PostgreSQL was `16.4 (Debian 16.4-1.pgdg110+2)`. The table's `pg_class`
estimate was zero even though 25 temporary rows were present; `EXPLAIN`
estimated 14 rows for each scan and found 25. Both geometry and geography GiST
indexes existed. The planner chose sequential scans at this tiny cardinality,
which is expected and does not establish index behavior at scale.

| Query | Plan | Estimated / actual rows | Execution | Shared buffers / spill |
| --- | --- | ---: | ---: | --- |
| Map bounds + intersection | Sequential scan, in-memory quicksort, limit | 14 / 25 | 0.872 ms | 5 shared hits at root; 0 reads; no temp I/O |
| Catalogue count in bbox | Sequential scan + aggregate | 14 / 25 scanned | 0.025 ms | 2 shared hits; 0 reads; no temp I/O |
| Catalogue page projection/order | Sequential scan, in-memory quicksort, limit | 14 / 25 | 1.341 ms | 79 shared hits at root; 0 reads; no temp I/O; 37 KiB sort |

No GiST index condition was used. At 25 rows this is not a negative index
finding. A representative-cardinality plan remains required before drawing
index or capacity conclusions. The query script also received an explicit
`text` cast for nullable optional filters after the first read-only capture
found PostgreSQL could not infer the type of an untyped `NULL`; the corrected
tool produced all three plans above.

## Worker, broker, and object-storage correlation

During the 30-second queue run, Prometheus was queried for its most recent
samples and then for the ten-minute maximum. The import queue reached a sampled
depth of 1 and the oldest queued age reached 0.58 seconds; processing-run gauge
was 0. JetStream metrics poller health was 1. Across the configured
`MYOTA_EVENTS` consumers, the sampled ten-minute maximum pending count,
ack-pending count, redeliveries, and oldest-message age were all zero. The
latest sample was 6.7 seconds old when freshness was checked. These results
show no observed sustained broker backlog at one import per second; they do
not establish maximum worker throughput.

The production SeaweedFS process exposes S3 counters and request histograms on
its private port 9324 metrics endpoint. A direct live check confirmed those
metric families; port 9327 is not listening in the deployed build. The
Collector had incorrectly targeted that unsupported second port. Deploy
commit [`3dc263a`](https://github.com/myota-platform/myota-deploy/commit/3dc263aae48c8cdfdaac6ae440dfda541a2eb1f7)
corrects the scrape, alert selectors, and dashboard labels. Its workflow and
Fleet reconciliation completed. Prometheus now records the SeaweedFS S3
counters and latency histograms from the corrected 9324 target. A bounded
one-VU upload was correlated with those metrics and post-cleanup worker/broker
gauges; see the [sanitized storage correlation artifact](phase0-storage-correlation-2026-10-08.md).
This is not a capacity test: object-storage saturation and sustained worker
capacity remain unmeasured.

An earlier capture showed the gateway using a development environment label
and route labels containing identifiers. The current Prometheus sample after
redeployment reports `deployment_environment="production"` and stable route
templates, including `/v1/geodata/import-uploads/{uploadId}/complete`,
`/v1/geodata/import-uploads/{uploadId}/parts/{partNumber}`, and
`/v1/geodata/load-test-runs/{testRunId}`. Route/environment telemetry is now
verified for the bounded upload workflow, with request records split between
gateway and geodata service.

## Conclusion and representative-scale gate

The five bounded write-profile executions and the empty-catalogue read
baseline are delivered and retained. The representative query/load review at
2,875 actual fixture entities is now documented in the
[representative query review](phase0-representative-query-review-2026-10-08.md).
The map query used its GiST index; the ordered catalogue page used its sort
index; no disk reads or sort spill occurred in the warm-cache capture. The map
row estimate was materially low, which is recorded for follow-up. The bounded
2-VU/60-second read run had zero failed requests and 28.75 ms p95. Worker and
JetStream backlog gauges returned to zero. SeaweedFS was observed handling a
small number of import-object requests without a storage signal indicating a
bottleneck. These observations close the current Phase 0 evidence gate at the
measured dataset size; they do not qualify capacity at larger entity/user/QSO
counts or under cold-cache/high-concurrency conditions.

SeaweedFS S3 metrics were configured against a non-listening 9327 endpoint;
this is corrected and verified in Prometheus after a bounded upload. Earlier
gateway environment and route-cardinality issues are also corrected; the live
sample now shows production labels and stable endpoint templates.

### Scale-gate implementation status — 8 October 2026

The API-based fixture provisioner defines 10,000 synthetic point inputs in
four Sevilla imports of 2,500, in a dedicated category not assigned to a
programme. They are clearly labelled synthetic and are not real parks. The
user confirmed the permanent input size and then directed that only 5% per
import be promoted: 2.5% to `CANDIDATE` and 2.5% to `APPROVED`; the remainder
is rejected from staging. Since 2.5% of a 2,500-feature import is fractional,
the implementation alternates 62/63 records by status, producing 250 of each
status in a fresh four-import run. The guarded provisioner has no cleanup
mode. See the
[fixture provisioner](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/provision_scale_fixtures.py).

#### Provisioning exception and current progress

Before the 5% split was requested, the first 2,500-feature import had already
been queued to promote every record as `APPROVED`. The worker completed it
before the provisioner was stopped. As of the read-only verification at
`2026-10-08T13:34Z`, that import contained 2,500 approved entities and its
processing queue was `COMPLETED`. The import was then finalized through the
public API; the response reported 2,500 processed and 2,500 staged records
discarded. Approved entities cannot be downgraded under current lifecycle
rules, so this first batch is retained as an explicit exception rather than
retired or deleted.

The revised provisioner recognized the exact 2,500-entity legacy prefix and
completed the remaining three imports at the requested split. The final
catalogue contains 2,875 permanent fixtures: 2,500 approved from the
pre-change batch, plus 375 selected across the remaining imports (187
Candidate and 188 Approved). The remaining 7,500 input features were
preprocessed and finalized; rejected/processed staged records were discarded.
This is not equivalent to a fresh-run 5% sample, so the exception is retained
in every result summary.

Source-provided administrative location fields are now retained as
`SOURCE_DATA` and skip redundant reverse-geocoder requests when a country code
is present. The map query-plan script now matches the live bbox SQL, including
the intersection and optional programme/status filters. Gateway telemetry
uses stable route templates and the deployment environment. SeaweedFS metrics,
scrapes, dashboard and alerts are implemented. GitHub quality and chart-render
checks passed. The first Fleet rollout attempted to start a second SeaweedFS
process on the single-writer PVC; its filer LevelDB lock rejected the new pod
while the old pod remained healthy. Helm now uses `Recreate`; the replacement
pod reached Ready and its live port 9324 endpoint returned SeaweedFS S3
counters and histograms. The Collector target and dashboard selectors are
corrected, and Helm hashes the rendered Collector config into the pod template
so Fleet ConfigMap changes trigger its rollout automatically. Fleet applied
the correction; Prometheus recorded the expected bucket operations during the
bounded upload and worker/broker gauges were zero after cleanup. Details are in
the [storage correlation artifact](phase0-storage-correlation-2026-10-08.md).

**Phase 0 representative-evidence gate: complete at 2,875 entities.** The
pre-fixture five-row planner estimate and zero-match plans are retained only
as a historical baseline in the
[pre-fixture plan snapshot](phase0-current-catalogue-plan-2026-10-08.md).
The completed post-fixture query and bounded-read review is linked above. No
test changed service replicas or injected faults. This gate does not claim
capacity at the original 10,000 promoted-entity level or at broader user/QSO
loads; those remain later scaling work.
