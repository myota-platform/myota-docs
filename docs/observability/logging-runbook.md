# Structured logging operations runbook

**Status:** Identity LogQL queries are qualified against Loki OTLP structured
metadata; Tempo stores the matching trace. Helm revision 225 mounted the
corrected datasource and dashboard, Grafana logged successful provisioning,
and all seven mounted dashboard queries returned successfully against live Loki.
Fleet release readiness and authenticated UI link navigation are still being
verified. Worker correlation, retention, capacity, redaction, and database gates
remain open.

Structured logs travel from services through the OpenTelemetry Collector to Loki.
PostgreSQL and application state remain authoritative; logs are diagnostic
telemetry and are not an audit archive.

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

The Deploy-owned datasource config links the `trace_id` structured-metadata field to Tempo using Grafana's label-type derived field. The matching Identity trace exists in Tempo, but the corrected provisioning must roll out before the UI link itself is accepted.

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

As of 2026-10-11, Loki and the Collector are Ready, but the live service images
have not advanced to all logging commits. Geodata image publication is blocked
by its JetStream worker integration test failures. No live request-to-worker
correlation query, retention expiry, object-growth baseline, or Compose parity
acceptance has been recorded. See the
[implementation evidence](evidence/logging-implementation-2026-10-11.md) and
[phase plan](logging-implementation.md).
