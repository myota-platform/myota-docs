# Geodata load tests and query evidence

This runbook defines bounded workloads and database evidence for the
[geodata horizontal-scaling roadmap](../horizontal-scaling-roadmap.md).
The current policy designates the live K3s environment on `spainip.es`, reached
through `https://api.myota.top`, as **provisional production and the required
target for all load/performance qualification**. Production-targeted test
results are provisional capacity evidence for this specific deployment, not a
general production-readiness guarantee. Run one profile at a time, use a
dedicated test account, retain exact-tag cleanup, stop on unexpected errors or
resource pressure, and preserve every harness acknowledgement and hard cap.

This policy applies to load and performance tests only. Unit/integration tests
and destructive failure-injection, restart, or chaos exercises remain isolated
from the live deployment; they must run in CI or a separate test environment.
Production runs are manually initiated, never run by CI, and require the
explicit hostname/environment acknowledgement described below.

## Workload profiles

The implementation lives in
[`myota-geodata-service/loadtests/geodata-workloads.js`](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/geodata-workloads.js).
It uses Grafana k6, available on macOS through `brew install k6`.

| Profile | Representative operation | Fixture/result |
| --- | --- | --- |
| `large-upload` | Resumable GeoJSON session and bounded binary parts; one real upload iteration per VU | Upload/preprocess run tagged with a unique run ID; defaults to about 3–4 MiB per file and is capped around 23 MiB |
| `simultaneous-edits` | Concurrent PATCH edits against one shared entity | Test-owned entity; edits remain isolated to its run tag |
| `preprocessing` | Repeated bounded dataset submissions | Imports remain available for validation; no promotion |
| `promotion` | Preprocess, validate, and enqueue selected candidates | Exercises the approval queue and creates tagged approved entities |
| `queue-backlog` | Steady accepted submissions without waiting on workers | Builds a bounded import-worker queue for backlog observation |

### Current harness caps

The limits below are read from the current harness and apply regardless of
target environment. The harness currently permits at most 50 VUs and a
duration of 1 second through 10 minutes. Defaults are 3 VUs for
`large-upload`, 8 for the other write profiles, and 2 minutes. Profile-specific
caps are 5,000 features for `large-upload` (2,500 by default), one for
`simultaneous-edits`, 50 for `queue-backlog`, 25 for `promotion`, and 100 for
`preprocessing`. Imports per VU are capped at 30 for `queue-backlog`, 2 for
`promotion`, and 5 for the other profiles. Per-feature padding is capped at
4 KiB, and `queue-backlog` is limited to one submission per second. Uploads
make one upload iteration per VU and send parts no larger than 16 MiB.

These are the implementation's hard bounds, not recommended operating
settings. For provisional-production qualification, begin at the lowest
meaningful profile size and increase only after checking service health and
the corresponding Grafana signals. Preserve all guards.

Non-production runs require all three protections:

1. `MYOTA_ENV` must be `development`, `test`, or `staging`.
2. The target must be a local, `.test`, or `.local` hostname, or an exact
   hostname explicitly listed in `MYOTA_LOAD_TEST_ALLOWED_HOSTS` for an
   approved non-production staging environment. Known production hosts and
   the `spainip.es` domain are rejected independently of that allowlist.
3. `MYOTA_LOAD_TEST_ALLOW_NONPROD=YES` must be supplied explicitly.

Production mode has separate opt-ins and exact-host allowlisting, but it uses
the same code-level caps shown above. Under the current policy, only
`api.myota.top` is the qualification target; do not redirect production runs
to another cluster and call them equivalent. Use a dedicated global-admin
test account; never
put credentials in the repository or command history. Production tests should
use a dedicated test administrator, not an everyday operator account.

Uploads use the existing `/v1/geodata/import-uploads` session API, not the
disabled single-request `/imports/upload` route. SHA-256 checks protect the
whole fixture and each raw part; client parts are bounded at 16 MiB. Completion
reads `importRun.id`, reconciles an uncertain response against session state,
and aborts incomplete sessions. Duration is the upload scenario's maximum
budget, not an idle looping period. See the
[upload verification and recovery record](load-test-upload-verification.md)
for regression tests and local results.

Example against the local gateway:

```bash
MYOTA_ENV=development \
MYOTA_LOAD_TEST_ALLOW_NONPROD=YES \
MYOTA_BASE_URL=http://localhost:8090 \
MYOTA_LOAD_TEST_EMAIL="$MYOTA_TEST_ADMIN_EMAIL" \
MYOTA_LOAD_TEST_PASSWORD="$MYOTA_TEST_ADMIN_PASSWORD" \
MYOTA_LOAD_TEST_PROFILE=preprocessing \
k6 run myota-geodata-service/loadtests/geodata-workloads.js
```

Production example (select one profile and execute it only during an approved
window):

```bash
MYOTA_ENV=production \
MYOTA_LOAD_TEST_ALLOW_PRODUCTION=YES \
MYOTA_LOAD_TEST_PRODUCTION_HOSTS=api.myota.top \
MYOTA_BASE_URL=https://api.myota.top \
MYOTA_LOAD_TEST_EMAIL="$MYOTA_PRODUCTION_TEST_ADMIN_EMAIL" \
MYOTA_LOAD_TEST_PASSWORD="$MYOTA_PRODUCTION_TEST_ADMIN_PASSWORD" \
MYOTA_LOAD_TEST_PROFILE=simultaneous-edits \
k6 run myota-geodata-service/loadtests/geodata-workloads.js
```

`MYOTA_LOAD_TEST_PROFILE` selects one of five workload profiles. The
`cleanup-only` recovery mode deletes the run named by
`MYOTA_LOAD_TEST_RUN_ID` without starting a workload. Optional controls
include `MYOTA_LOAD_TEST_VUS`, `MYOTA_LOAD_TEST_DURATION`,
`MYOTA_LOAD_TEST_FEATURES`, `MYOTA_LOAD_TEST_PADDING_BYTES`, and
`MYOTA_LOAD_TEST_IMPORTS_PER_VU`; the service
and harness reject values above their safety caps. Set
`MYOTA_LOAD_TEST_ALLOWED_HOSTS` only for an approved non-production staging
host; production uses the separate exact `MYOTA_LOAD_TEST_PRODUCTION_HOSTS`
allowlist.

After an import POST is accepted, the asynchronous upload handoff may briefly
return 404 from `GET /v1/geodata/imports/{runId}` before the durable run record
is visible. The harness retries that specific status-poll 404 and excludes it
from `http_req_failed`; other statuses, including 5xx responses, remain
failures. If polling times out, it reports the final HTTP status and sanitized
API details. This preserves the request-failure threshold while accounting for
the documented registration handoff.

For production write profiles, latency and request-failure thresholds are
report-only during execution and evaluated after the configured workload
finishes. This prevents an early threshold abort from leaving accepted imports
in flight before teardown. Final threshold failures still make the k6 run fail;
non-production profiles retain early-abort behavior.

Latency is tagged by request class and operation. The production 2-second p95
applies to API/control requests, not bulk part transfer or fixture cleanup.
Large-upload parts have a separate 60-second p95 ceiling (parts are capped at
16 MiB); their duration is also reported as its own k6 sub-metric. This keeps
network and object-storage transfer time visible without treating it as an API
control-plane response-time regression. Cleanup remains separately tagged and
does not distort the workload latency threshold.

## Fixture cleanup

Each import's source metadata and each created entity's provenance carry the
unique `loadTestRunId`. k6 teardown invokes
`DELETE /v1/geodata/load-test-runs/{testRunId}` and retries while a tagged job is
still active. The cleanup endpoint is disabled unless
`MYOTA_LOAD_TEST_CLEANUP_ENABLED=1` and `MYOTA_ENV` is explicitly declared.
Production additionally requires `MYOTA_LOAD_TEST_ALLOW_PRODUCTION_CLEANUP=YES`.
Local Compose enables cleanup; Helm defaults it off. For a production test
window, set `geodataLoadTestCleanup.enabled=true` and
`geodataLoadTestCleanup.allowProductionCleanup=true` while
`auth.environment=production`. Verify the deployment is ready before running
the test. After teardown confirms cleanup, turn both options off and redeploy
immediately. If cleanup fails, stop further writes and resolve the tagged run
before proceeding.

The endpoint also requires the identity service's global-administrator role
(`GLOBAL_OPERATOR`; legacy `GLOBAL_ADMIN` tokens are also accepted) and the exact
confirmation `DELETE LOAD TEST DATA <testRunId>`. It refuses active uploads/imports or
promotion queues, and checks every generated entity for linked activations,
QSOs, or award progress before deletion. On success it removes the tagged
source objects from the geodata-import bucket, upload spool files, staged
candidates, processing queue records, geodata entities, import runs, and local
outbox records. Terminal upload sessions and their part metadata are removed
by exact source tag, with `uploadSessionsDeleted` in the response; session and
import references to the same object are deleted once. Active upload sessions
are refused, not force-aborted. It records one cleanup event. Events already published to
JetStream cannot be recalled by this database cleanup and expire according to
the configured stream-retention policy.

If k6 is interrupted, do not start another profile until the run has been
cleaned up. Reuse its run ID, sign in with the dedicated target-environment
admin, and submit the exact confirmation to the endpoint. Cleanup intentionally fails
closed if the activity API cannot verify that entities have no protected
activity.

In Kubernetes, ensure the geodata deployment's `MYOTA_ACTIVITY_URL` resolves to
the Activity Service. The MyOTA Helm chart sets it from
`services.activity.internalUrl` (default `http://myota-activity:8004`). Cleanup
fails closed before deleting fixtures when this cross-service safety check
cannot reach the activity API.

The harness's `cleanup-only` mode performs only the tagged-run cleanup and
prints sanitized API problem details and correlation IDs if the endpoint fails.
Use it to recover a failed teardown; do not start another workload until it
reports successful cleanup. Reuse the failed run's exact identifier, for
example `MYOTA_LOAD_TEST_PROFILE=cleanup-only`
`MYOTA_LOAD_TEST_RUN_ID=lt-20261006123955-simultaneous-edits`, with the same
target and credentials used for the original run.
The harness treats only the endpoint's explicit active-upload/import/promotion
responses as retryable during cleanup, excludes those waits from failed-request
thresholds, and allows up to six minutes for cleanup; other client errors fail
immediately. Failed import submissions print the HTTP status and bounded,
credential-redacted problem response, including request/correlation IDs when
available. Cleanup retry waits print the safe API detail and attempt number.
Use these diagnostics to identify the service-side cause; do not weaken the
workload acceptance or request-failure thresholds.
After an interrupted resumable transfer, abort only the logged upload ID
through its user-bound upload DELETE endpoint, then retry exact-tag cleanup.
The service's current authority is relational: import/entity updates use
request transactions and optimistic revisions, not shared JSON snapshots.
See the [relational authority rollout](../phase1-relational-authority.md)
and its concurrency regression coverage.

## Query-level timing and plans

API route duration measures the full request; it cannot show whether time was
spent in PostGIS, object storage, or another API. The geodata service therefore
also exports:

- `myota_geodata_postgis_query_duration_seconds{query="bbox"}` for the indexed
  bounding-box/intersection query used by map reads;
- the same histogram labeled `query="entity_upsert"` around relational entity
  upsert work; and
- `myota_geodata_slow_queries_total{query=...}` when a measured query exceeds
  `MYOTA_SLOW_QUERY_THRESHOLD_MS` (250 ms by default).

For an actual PostgreSQL execution plan, use
[`geodata-query-plan-evidence.py`](https://github.com/myota-platform/myota-geodata-service/blob/main/scripts/geodata-query-plan-evidence.py)
against the current provisional-production PostGIS database for qualification.
It requires `MYOTA_ENV=production`,
`MYOTA_ALLOW_EXPLAIN_ANALYZE=YES`,
`MYOTA_ALLOW_PRODUCTION_EXPLAIN=YES`, and an exact database-host match in
`MYOTA_PRODUCTION_GEO_DATABASE_HOSTS`. Non-production mode remains available
for development checks, but does not qualify the production capacity gates.
The script sets the session read-only, applies a 5-second default statement
timeout (hard maximum 30 seconds), and bounds the requested extent and row
limit. Its
JSON output contains the timestamp, environment/database host, server version,
estimated entity cardinality, GiST index definitions, and
`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`
plans for the map-bounds query, the paged entity-catalogue count, and the
catalogue page projection/order. Review the output before sharing because plan
details reveal schema and index metadata. The catalogue page uses the same
bounding-box filter, `ORDER BY lower(name), id`, page limit, and projected
entity fields as the relational catalogue implementation; it is not a generic
stand-in query.

Example:

```bash
MYOTA_ENV=development \
MYOTA_ALLOW_EXPLAIN_ANALYZE=YES \
GEO_DATABASE_URL="$MYOTA_NONPROD_GEO_DATABASE_URL" \
python3 myota-geodata-service/scripts/geodata-query-plan-evidence.py \
  --bbox=-5.99,37.37,-5.90,37.43 \
  --output /tmp/myota-postgis-query-plan.json
```

For production, run the tool from the geodata API pod or another controlled
host with network access to the production PostGIS service. Do not put the
database URL in a command argument or print it. The exact host allowlist must
contain only `myota-geo-postgis` for the current K3s deployment. Example
environment acknowledgements (the database URL comes from the pod's existing
secret-backed environment and is not echoed):

```bash
MYOTA_ENV=production \
MYOTA_ALLOW_EXPLAIN_ANALYZE=YES \
MYOTA_ALLOW_PRODUCTION_EXPLAIN=YES \
MYOTA_PRODUCTION_GEO_DATABASE_HOSTS=myota-geo-postgis \
python3 scripts/geodata-query-plan-evidence.py \
  --bbox=-5.99,37.37,-5.90,37.43 --limit=250
```

The report is evidence, not a tuning conclusion: run it with representative
provisional-production cardinality and selectivity, inspect whether the spatial and
catalogue indexes are used, compare buffers and actual rows to estimates, and
retain the report with the associated workload summary. Record each plan's
planning/execution time, node type and index condition, estimated versus
actual rows, loops, shared hit/read blocks, and any sort/hash spill. Do not
infer service capacity from one plan or a tiny fixture.
Each statement is read-only and subject to a 5-second default statement
timeout, adjustable only up to 30 seconds with `--statement-timeout-ms`; the
output records the selected timeout. Whole-table cardinality is taken from the
planner's row estimate rather than running an unbounded exact `count(*)`.

## Phase 0 evidence review status

The sanitized **[8 October production evidence record](phase0-production-evidence-2026-10-08.md)** contains the
run IDs, profile limits and results, cleanup receipts, baseline summaries,
query-plan review, and measured telemetry limits. All five bounded write
profiles passed against the designated `myota` K3s deployment on `spainip.es`
(`api.myota.top`) and cleaned their tagged data. Earlier non-production
summaries remain historical and do not qualify these current gates.

The empty-catalogue read run passed health and paged-list checks; a separate
read-only run used five temporary approved test geometries to exercise map and
detail reads, then the source promotion run removed them. Neither run created
permanent seed entities. Those runs were small-cardinality checks. The
permanent synthetic Sevilla input set (10,000 features in four 2,500-feature
imports) is now finalized in the isolated `SCALE_TEST_FIXTURE` category,
unassigned from all programmes and with no automatic cleanup. Three imports
used the requested 5% promotion sample (2.5% Candidate, 2.5% Approved); the
first import had already been queued for full approval and is a documented
exception. The final catalogue contains 2,875 synthetic entities. The complete
sanitized result and limitations are in the [representative query review](phase0-representative-query-review-2026-10-08.md).

The guarded production plan tool produced map, catalogue-count, and
catalogue-page plans both before and after fixture provisioning. The pre-fixture
25-row sequential-scan snapshot is historical; at 2,875 entities, the map/count
queries used the spatial GiST index and the catalogue page used its sort index.
See the detailed evidence artifact for estimates, buffers, timings, and the map
row-estimation gap.

During the 30-import/30-second backlog profile, the ten-minute Prometheus
window recorded a maximum import queue depth of 1 and maximum queued age of
0.58 seconds. JetStream pending, ack-pending, redeliveries, and oldest-message
age stayed at zero in the sampled series. This shows no observed sustained
broker lag at one import per second, not maximum worker capacity.

The deployed SeaweedFS build exposes both master and S3 metrics on private
port 9324. A live check confirmed S3 counters and histograms there; port 9327
does not listen. Deploy commit
[`3dc263a`](https://github.com/myota-platform/myota-deploy/commit/3dc263aae48c8cdfdaac6ae440dfda541a2eb1f7)
corrects the Collector target, alert selectors, and dashboard labels to use
the verified endpoint. GitHub validation and Fleet reconciliation passed; a
post-correction Prometheus sample confirms the expected bucket operations. The
chart hashes the Collector scrape configuration into its pod template, so
Fleet ConfigMap changes trigger an automatic Collector rollout. A bounded
one-VU large-upload profile was correlated with SeaweedFS and post-cleanup
worker/broker metrics; its sanitized summary and limitations are in the
[storage correlation artifact](phase0-storage-correlation-2026-10-08.md).
The latest read-only production plan estimated five rows but returned none
from the ordered catalogue scan, so it is explicitly not scale evidence; see
the [sanitized plan snapshot](phase0-current-catalogue-plan-2026-10-08.md).
The gateway now emits stable route templates and takes `MYOTA_ENV` from the
chart. A live Prometheus sample now confirms production environment labels and
identifier-free route templates for the upload and cleanup endpoints.

**Phase 0 evidence gate is complete at the measured 2,875-entity catalogue.**
The representative plans, bounded read profile, gateway route/environment
labels, SeaweedFS operation samples, and worker/broker signals are reviewed in
the linked evidence. Do not generalize these results to 10,000 promoted
entities or larger user/QSO loads, and do not infer capacity from zero sampled
broker lag or end-to-end upload timing alone.

## JetStream consumer lag

The geodata service polls JetStream consumer state directly (default stream
`MYOTA_EVENTS`, 15-second interval, maximum 100 consumers per stream). Metrics
include unconsumed pending count, ack-pending count, the broker's redelivery
count, and oldest outstanding message age. The poller reads the stream's
stored timestamp for the sequence immediately after the consumer acknowledgement
floor (or delivered sequence when only undelivered messages remain); it does
not consume or acknowledge messages. If retention purges that sequence or the
sequence subject does not match a consumer filter, the age-availability gauge
is zero rather than reporting an invented age. Configure stream names with
`MYOTA_JETSTREAM_METRICS_STREAMS` if the deployment uses additional streams.

The Grafana dashboard **MyOTA JetStream backlog and PostGIS query performance**
contains consumer pending, ack-pending, redeliveries, oldest age and age
availability, poller health, and dedicated PostGIS latency/slow-query panels.
It is provisioned in both local Compose and Helm. These broker metrics
complement—not replace—the database outbox and import-run measurements. The current broker poller and dashboard are planned to be replaced by Surveyor; the PostGIS panels remain. See the [consolidation roadmap](../../observability/nats-surveyor-migration.md).
