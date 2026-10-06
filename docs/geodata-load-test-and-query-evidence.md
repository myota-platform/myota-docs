# Geodata load tests and query evidence

This runbook defines bounded write workloads and database evidence for the
[geodata horizontal-scaling roadmap](geodata-horizontal-scaling-roadmap.md).
Write profiles can target production only through explicit hostname and
environment acknowledgement, and production fixture cleanup must be enabled
separately. Production execution is operator-controlled and is never started
by CI.

## Workload profiles

The implementation lives in
[`myota-geodata-service/loadtests/geodata-workloads.js`](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/geodata-workloads.js).
It uses Grafana k6, available on macOS through `brew install k6`.

| Profile | Representative operation | Fixture/result |
| --- | --- | --- |
| `large-upload` | Multipart GeoJSON upload; one upload per VU | Upload/preprocess run tagged with a unique run ID; defaults to about 3–4 MiB per file and is capped around 23 MiB |
| `simultaneous-edits` | Concurrent PATCH edits against one shared entity | Test-owned entity; edits remain isolated to its run tag |
| `preprocessing` | Repeated bounded dataset submissions | Imports remain available for validation; no promotion |
| `promotion` | Preprocess, validate, and enqueue selected candidates | Exercises the approval queue and creates tagged approved entities |
| `queue-backlog` | Steady accepted submissions without waiting on workers | Builds a bounded import-worker queue for backlog observation |

Non-production runs require all three protections:

1. `MYOTA_ENV` must be `development`, `test`, or `staging`.
2. The target must be a local, `.test`, or `.local` hostname, or an exact
   hostname explicitly listed in `MYOTA_LOAD_TEST_ALLOWED_HOSTS` for an
   approved non-production staging environment. Known production hosts and
   the `spainip.es` domain are rejected independently of that allowlist.
3. `MYOTA_LOAD_TEST_ALLOW_NONPROD=YES` must be supplied explicitly.

Production runs instead require `MYOTA_ENV=production`,
`MYOTA_LOAD_TEST_ALLOW_PRODUCTION=YES`, and an exact hostname entry in
`MYOTA_LOAD_TEST_PRODUCTION_HOSTS`. Known production hosts still have to be
explicitly listed. Existing per-profile hard caps remain in force in
production: 10 minutes, 20 VUs (8 uploads), feature and import caps, 4 KiB
maximum padding per feature, and at most one queue-backlog submission per
second. These are intentional safety bounds, not configurable away for a
production target.

The harness caps duration at 10 minutes, ordinary profiles at 20 VUs, uploads
at 8 VUs, features per import at 100 (50 for queue backlog; 25 for promotion;
5,000 for uploads), and submissions at five per VU (30 for queue backlog; two
for promotion). Upload features include 1 KiB synthetic
padding by default, adjustable up to 4 KiB per feature, so the file exercises
transfer/storage as well as geometry parsing. The steady backlog profile is
capped at one submission per second.
Use a dedicated global-admin account created for the target environment; never
put credentials in the repository or command history. Production tests should
use a dedicated test administrator, not an everyday operator account.

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
outbox records. It records one cleanup event. Events already published to
JetStream cannot be recalled by this database cleanup and expire according to
the configured stream-retention policy.

If k6 is interrupted, do not start another profile until the run has been
cleaned up. Reuse its run ID, sign in with the dedicated target-environment
admin, and submit the exact confirmation to the endpoint. Cleanup intentionally fails
closed if the activity API cannot verify that entities have no protected
activity.

The harness's `cleanup-only` mode performs only the tagged-run cleanup and
prints sanitized API problem details and correlation IDs if the endpoint fails.
Use it to recover a failed teardown; do not start another workload until it
reports successful cleanup. Reuse the failed run's exact identifier, for
example `MYOTA_LOAD_TEST_PROFILE=cleanup-only`
`MYOTA_LOAD_TEST_RUN_ID=lt-20261006123955-simultaneous-edits`, with the same
target and credentials used for the original run.

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
against a development, test, or staging database. It requires
`MYOTA_ENV=development|test|staging` and
`MYOTA_ALLOW_EXPLAIN_ANALYZE=YES`, rejects production-like database hostnames,
sets the session read-only, and bounds the requested extent and row limit. Its
JSON output contains the timestamp, environment/database host, server version,
row count, GiST index definitions, and `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`
plan. Review the output before sharing because plan details reveal schema and
index metadata.

Example:

```bash
MYOTA_ENV=development \
MYOTA_ALLOW_EXPLAIN_ANALYZE=YES \
GEO_DATABASE_URL="$MYOTA_NONPROD_GEO_DATABASE_URL" \
python3 myota-geodata-service/scripts/geodata-query-plan-evidence.py \
  --bbox=-5.99,37.37,-5.90,37.43 \
  --output /tmp/myota-postgis-query-plan.json
```

The report is evidence, not a tuning conclusion: run it with representative
non-production cardinality and selectivity, inspect whether the spatial index
is used, compare buffers and actual rows to estimates, and retain the report
with the associated workload summary.

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
complement—not replace—the database outbox and import-run measurements.
