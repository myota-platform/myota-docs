# Structured logging operations runbook

The logging stack is deployed through the existing OpenTelemetry Collector,
Loki, Tempo, Prometheus, and Grafana services. This runbook covers bounded
application and worker telemetry. PostgreSQL remains authoritative for domain
and job state; logs are diagnostic evidence, not an audit archive.

As of 12 October 2026, alert rules link here using the
[`runbook_url`](#alert-response) annotation. Read the
[latest evidence](evidence/logging-implementation-2026-10-12.md) for current
deployment state and acceptance limits. Do not infer capacity from the
single-node sample.

## Query examples

Run these in Grafana Explore with the **MyOTA Loki** datasource. The examples use `service_namespace` as the indexed Loki label. OTLP fields such as `severity_text`, `trace_id`, `event`, and `correlation_id` are Loki structured metadata, not JSON in the log line; filter them directly without a `| json` parser. Do not turn request IDs, trace IDs, correlation IDs,
URLs, user/domain IDs, or exception messages into Loki labels.

### Service errors

```logql
{service_namespace="myota"}
  | severity_text="ERROR"
```

Narrow to one service/component:

```logql
{service_namespace="myota"}
  | service_name="myota-geodata"
  | component="api"
  | severity_text=~"ERROR|FATAL"
```

Inspect recent warn/error volume without creating high-cardinality labels:

```logql
sum by (service_name, severity_text) (
  rate({service_namespace="myota"} | severity_text=~"WARN|ERROR" [5m])
)
```

### Reconstruct an asynchronous workflow

Use the stable business `correlation_id` when the originating envelope provides
one. A missing correlation ID is an evidence gap; do not substitute a trace ID
or synthesize a new workflow ID during investigation.

```logql
{service_namespace="myota"}
  | correlation_id="<correlation-id>"
```

Filter worker lifecycle records:

```logql
{service_namespace="myota"}
  | correlation_id="<correlation-id>"
  | event=~"job.received|job.started|job.completed|job.retry|job.failed"
```

Inspect repeated retry or permanent-failure classifications:

```logql
{service_namespace="myota"}
  | event=~"job.retry|job.failed"
  | error_type!=""
```

Workers intentionally log bounded metadata, IDs, attempts and error classes.
They do not emit full payloads or raw exception text.

### Navigate between logs and traces

- In Loki, open a record with a non-empty `trace_id` and follow the provisioned
  Tempo derived-field link.
- In Tempo, use the provisioned trace-to-logs action to query Loki with the
  trace ID, service name and time window.
- A trace ID joins one distributed execution. A business `correlation_id`
  joins asynchronous work across separate traces and retries.

The Grafana datasource configuration links the `trace_id` structured-metadata
field to Tempo using a label-type derived field. The 12 October production
snapshot recorded the corrected provisioning at Helm revision 280. Testing
revision 301 also contains the Loki/Tempo datasources and dashboard
provisioning. Authenticated browser navigation in both directions remains
unverified; do not treat the provisioned link as proof of an authorized UI
path.

### Provisioned Grafana dashboards

The current dashboard definitions are owned by the monorepo deployment tree
and are provisioned in the Grafana **MyOTA** folder.

#### MyOTA Logs / Failures (`myota-logs-failures`)

This dashboard uses MyOTA Loki (`myota-loki`) to show log volume,
warnings/errors, worker retries, failed HTTP completions, and log records for
a correlation ID. Its `service`, `environment`, `severity`, `component`, and
`correlation_id` variables constrain the queries. The [Compose source](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/observability/grafana/dashboards/myota-logs-failures.json)
is packaged for Helm at
[`deploy/helm/myota/observability/grafana/dashboards/myota-logs-failures.json`](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/helm/myota/observability/grafana/dashboards/myota-logs-failures.json).

#### MyOTA Distributed Traces (`myota-traces`)

This dashboard uses MyOTA Tempo (`myota-tempo`) to search for error traces and
traces longer than one second. The one-second value is a query default, not an
SLO. Its service textbox defaults to all MyOTA services. The [Compose source](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/observability/grafana/dashboards/myota-traces.json)
is packaged for Helm at
[`deploy/helm/myota/observability/grafana/dashboards/myota-traces.json`](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/helm/myota/observability/grafana/dashboards/myota-traces.json).

Both dashboards default to the last 30 minutes and refresh every 30 seconds.
The Logs / Failures dashboard scopes queries to the active deployment namespace;
Compose uses `myota`, while Helm renders the release namespace into the
dashboard. The provisioner and dashboard folder definitions are in
[`deploy/observability/grafana/provisioning/dashboards/dashboards.yml`](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/observability/grafana/provisioning/dashboards/dashboards.yml)
and its packaged Helm copy.

For a log-to-trace investigation, set the dashboard time range first, narrow
service/environment/component as needed, then set `correlation_id` to the
stable workflow identifier. Open a matching log record with a non-empty
`trace_id` and use **View trace**. In Tempo, select an error or slow trace and
use its trace-to-logs action to return to Loki. A trace ID is scoped to one
distributed execution; use the business correlation ID to follow retries and
separate asynchronous executions. If a trace or matching log is absent, check
the selected time range, service filter, ingestion health, and configured
retention before treating the absence as proof that no work occurred.

The dashboards query bounded service, environment, severity, component, and
route dimensions. Trace IDs and correlation IDs are query fields, not indexed
Loki labels; participant identifiers, email addresses, request bodies, SQL,
and message payloads must not be added to labels or dashboard variables. The
HTTP completion panel intentionally includes only error
`http.request.completed` events. Loki's configured retention is 14 days;
Tempo is short-lived diagnostic storage, not an event archive. PostgreSQL,
outbox rows, and domain records remain authoritative for work and business
state.

**Testing deployment evidence (14 October 2026):** Fleet reported the
`myota-test` bundle Ready at monorepo commit `e00ba2960595601df03bd71c41f0641b7914197f`.
The Grafana provisioning logs reported dashboards finished provisioning; the
observability ConfigMap contained both dashboard files and Loki/Tempo
datasources. A read-only Tempo search returned 20 stored traces. At that
check, the failed-trace and over-one-second queries returned no matches, so
those panels were provisioned but had no matching sample. No synthetic trace
was written. See the [Testing deployment record](../operations/test-deployment.md)
and deployment work item [#22](https://github.com/myota-platform/myota/issues/22).

## Loki or Collector ingestion failure

1. Check the Collector Deployment/Pod and its logs for receiver, queue, retry,
   exporter, or backpressure errors.
2. Check Collector scrape/health signals in Prometheus. A missing target is
   unavailable telemetry, not zero application activity.
3. Check Loki readiness at `/ready`, its pod logs, and free space in the small
   working volume. Durable TSDB index/chunks belong in SeaweedFS bucket
   `myota-loki`.
4. Compare service logs with a bounded Loki query over the matching UTC window.
   The current system-wide dashboard and evidence use UTC.
5. Do not delete Loki bucket data to repair ingestion. Preserve object-store
   data and resolve the receiver/exporter/configuration failure.
6. Loki retention is owned by its 14-day compactor configuration. Do not add a
   separate bucket lifecycle rule or MyOTA cleanup worker.

## Safe capacity review

Record, over a representative period:

- bytes received by the Loki distributor per day;
- growth of the `myota-loki` SeaweedFS bucket;
- Loki working-volume utilization and compactor temporary-space use;
- Collector exporter queue utilization, retries and refused telemetry;
- Loki/Collector CPU and memory at observed peak ingestion.

Set capacity thresholds only from these measurements. The single-node bucket
and small working PVC do not imply measured headroom; do not infer a scale-out
threshold from an empty bucket or a short smoke test. Reassess single-binary
Loki when observed ingest, query latency, compaction duration, or object-store
capacity approaches the actual node/storage budget.

## Boundaries and current acceptance status

The worker logs do not replace database job/outbox records. Prometheus remains
the owner for availability, latency, rate and queue-depth alerts. Loki alerts
should be limited to discrete exceptional events only after the deployed
producer emits a reliable, privacy-safe event.

As of 2026-10-12, Identity, Programme, Activity, and Geodata request-to-worker
correlation paths have deployed evidence and their service-owned issues are
closed. Loki's live configuration reports 14-day retention and compactor
activity; actual expiry is not claimed. Representative capacity/recovery and
Compose runtime parity remain open in Deploy #6. See the
[12 October implementation evidence](evidence/logging-implementation-2026-10-12.md)
and [phase plan](logging-implementation.md).


## PostgreSQL and PostGIS telemetry status

PostgreSQL receiver metrics are active in the deployed Collector. Separate
least-privilege monitor roles expose bounded health and workload series for Core,
Activity, and Geo; live evidence records non-empty backend, database-size,
commit, rollback, and deadlock metrics, with authentication failures resolved.
The query-duration histogram remains Geodata-owned. Transaction-age, cache, and
vacuum/freeze panels remain unavailable rather than inferred. Database spans
exclude SQL statements, query text, and bind parameters. See the
[Operations retirement evidence](../operations/evidence/operations-service-retirement-2026-10-11.md)
and [current system state](../architecture/current-state.md).

The dashboard is
[`myota-postgresql-postgis.json`](https://github.com/myota-platform/myota/blob/e00ba2960595601df03bd71c41f0641b7914197f/deploy/observability/grafana/dashboards/myota-postgresql-postgis.json)
in the monorepo deployment tree; its packaged Helm copy is under
[`deploy/helm/myota/observability/grafana/dashboards/`](https://github.com/myota-platform/myota/tree/e00ba2960595601df03bd71c41f0641b7914197f/deploy/helm/myota/observability/grafana/dashboards).
It includes core connection/size/transaction panels and the existing bounded
Geodata PostGIS query-duration histogram. Empty database panels are expected
until the receiver is accepted.

## Alert response

All configured Prometheus and Grafana alert rules include a link to this
section. Alert conditions use bounded metric series. Do not create an alert
per request, correlation ID, entity, message, or job ID.

| Signal | First checks | Recovery boundary |
|---|---|---|
| Collector, Loki, or Tempo unavailable | Check Deployment readiness, Collector queue/retry logs, service endpoints, and persistent volume capacity. | Restore the telemetry component; application/database state remains authoritative. |
| API error rate or latency | Filter by fixed service, normalized route, method, and status; follow trace/log links when authenticated Grafana navigation is available. | Resolve the owning API or dependency fault; do not use request payloads as diagnostics. |
| Activity/Geodata worker failure, retry, or queue age | Filter logs by service and stable `correlation_id`; inspect the owning job row, durable pending state, and outbox relay metrics. | Recover or redrive from the owning database row only after cause is understood; keep handler idempotency. |
| Outbox age, retries, or unresolved dead letter | Inspect the service-owned outbox row, relay readiness, NATS connectivity, and bounded error class. | Database outbox remains the source of truth; do not delete an unresolved row to clear an alert. |
| PostgreSQL connection or collector signal | Check database readiness, the least-privilege monitor role, and pool/connection metrics. | Preserve business data; logs do not replace database health or recovery procedures. |
| NATS/JetStream capacity or Surveyor | Check server/Surveyor health and bounded stream/consumer metrics. | NATS is bounded delivery infrastructure, not the domain archive. Recover work from the service database. |

For every alert, keep the event body and labels free of payloads and secrets.
The only indexed Loki labels are `deployment_environment`,
`service_name`, and `service_namespace`; use structured metadata for
correlation and trace navigation.
