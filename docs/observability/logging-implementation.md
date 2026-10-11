Status: source, configuration, and the new logging images are implemented.
Helm has applied the new image digests and all Deployments are Ready, but its
revision 242 remains `pending-upgrade`; Fleet still reports the prior revision
241 as Ready with 67/67 resources. Treat the Helm release state as an open
rollout item until the controller reports a stable deployed revision.
PostgreSQL metrics and SQL-scrubbed Psycopg spans are active. Remaining
acceptance is listed below; this roadmap does not claim full completion.

[Current evidence and remaining gaps](evidence/logging-implementation-2026-10-11.md).
Parent: [MyOTA observability](overview.md)

Current evidence: [logging implementation evidence, 2026-10-11](evidence/logging-implementation-2026-10-11.md).

**Acceptance boundary:** checked deliverables below mean the source or deployment
asset is present and its stated focused check passed. They do not close a phase
whose live, Compose/Helm, capacity, privacy, or end-to-end exit evidence remains
open. See the linked evidence record for exact gates.

## Objective

Add durable, structured, correlated application and worker logging to the existing
Prometheus + Tempo + Grafana + OpenTelemetry Collector stack without introducing
a second observability control plane.

The target architecture is:

```mermaid
flowchart LR
  Services[HTTP services and workers]
  OTel[OpenTelemetry Collector]
  Prom[Prometheus]
  Tempo[Tempo]
  Loki[Loki]
  Grafana[Grafana]
  Alert[Alertmanager]

  Services -->|OTLP metrics| OTel
  Services -->|OTLP traces| OTel
  Services -->|OTLP structured logs| OTel
  OTel --> Prom
  OTel --> Tempo
  OTel --> Loki
  Postgres[PostgreSQL + PostGIS]
  Operations[Operations service
only for unsupported DB-specific telemetry]
  Postgres -->|PostgreSQL receiver metrics| OTel
  Postgres -->|database logs via OTLP/filelog| OTel
  Services -->|instrumented database client spans| OTel
  Postgres -.->|fallback for unsupported signals| Operations
  Operations -->|OTLP / existing scrape path| OTel
  Grafana --> Prom
  Grafana --> Tempo
  Grafana --> Loki
  Prom --> Alert
  Grafana --> Alert
```

MyOTA will use **Grafana Loki** as the log backend and the existing
**OpenTelemetry Collector** as the ingestion and processing layer. Promtail,
Filebeat, Fluent Bit, Elasticsearch, and OpenSearch are intentionally not part of
the first implementation.

## Design principles

1. Logs are structured telemetry, not unbounded `print()` output.
2. All services use the same resource identity and field semantics.
3. `trace_id`, `span_id`, `request_id`, and `correlation_id` have distinct purposes.
4. HTTP route labels use normalized route templates; identifiers never become metric
   or Loki labels.
5. Worker and NATS/JetStream operations are first-class observability subjects.
6. Payloads and secrets are not logged by default.
7. Metrics remain the primary source for rate, latency, availability, and queue alerts;
   logs are used for investigation and exceptional-event alerts.
8. Compose and Helm must remain functionally aligned.
9. Prefer direct OpenTelemetry collection/instrumentation for PostgreSQL and
   PostGIS telemetry; use the operations service only for database signals that
   cannot be collected safely and reliably through direct integrations.

## Correlation model

| Identifier | Scope | Purpose |
|---|---|---|
| `trace_id` | distributed execution path | OpenTelemetry trace correlation |
| `span_id` | one trace operation | link a log record to a specific span |
| `request_id` | one HTTP request | request/response troubleshooting |
| `correlation_id` | business/process workflow | correlate work across separate traces, queues and retries |
| `event_id` | one domain/message event | identify an emitted or consumed event |
| `job_id` | asynchronous job | follow job lifecycle and retries |

A single workflow may contain several traces while retaining one
`correlation_id`. For example an ADIF upload, JetStream processing, award
recalculation, and notification can remain queryable as one business operation
even if execution crosses asynchronous boundaries.

## Canonical structured log fields

Every application log record should support the following baseline fields:

```text
timestamp
severity
service.name
service.namespace = myota
service.instance.id
deployment.environment
component
event
trace_id
span_id
request_id
correlation_id
```

Context-specific fields may include:

```text
http.request.method
http.route
http.response.status_code
duration_ms

messaging.system
messaging.destination
messaging.message.id
messaging.operation
messaging.delivery_count

job.type
job.id
event.type
event.id

operator.id
programme.id
entity.id
import.id
activation.id
qso.id
award.id
```

Identifiers are fields/structured metadata, not Loki index labels.

## Loki label policy

Keep Loki labels deliberately low-cardinality. Initial labels should be limited to:

```text
service_name
service_namespace
deployment_environment
severity
component
```

A bounded `job_type` label may be added after validating cardinality.

Do **not** label on request IDs, trace IDs, correlation IDs, callsigns, operators,
entities, QSOs, imports, activations, programmes, awards, URLs, or arbitrary
exception messages.

## Data-handling policy

Do not log complete request/event payloads by default.

Explicitly exclude:

- Authorization headers and bearer tokens.
- JWTs, passwords, signing keys, S3 credentials, API keys and connection strings.
- Uploaded ADIF contents.
- GeoJSON/KML/GPX/WFS/ArcGIS bodies or complete geometries.
- Full user profiles or unrestricted personally identifying fields.
- Raw exception objects when they may embed credentials or payload fragments.
- Raw SQL literals, bind parameters, credentials, or unbounded query text in
  database spans, logs, metric labels, or diagnostic endpoints.

Prefer identifiers and bounded metadata such as import ID, feature count, source
type, object size, job type, delivery attempt, duration, result status, and
sanitized error classification.

## Retention baseline

Initial production target:

| Signal | Initial retention |
|---|---:|
| Prometheus | existing 15 days |
| Tempo | 7-14 days, capacity dependent |
| Loki | 14 days |
| DEBUG logs | disabled in production by default |

Loki retention is 14 days and is enforced by Loki's compactor. Loki's durable
TSDB index and chunks live in the dedicated SeaweedFS S3 bucket `myota-loki`,
using the existing SeaweedFS endpoint and S3 credential provisioning patterns.
Keep only a small local writable volume for active TSDB/WAL/index/cache and
compactor working data; it is not the authoritative log store. Do not provision
a 10-20 GiB Loki data PV. Do not add a separate MyOTA or SeaweedFS cleanup worker:
Loki's compactor owns expiration and deletion of retained log objects. Longer
WARN/ERROR retention may be considered later, but the first implementation
should keep one simple policy.

## Implementation phases

### Phase 1 - Shared structured logging foundation

Create a common MyOTA observability/logging helper and standardize resource
identity, severity, field names, redaction rules, and trace/span enrichment across
services.

Deliverables:

- [x] shared logging configuration;
- [x] JSON/structured log records;
- [x] OpenTelemetry logging provider/handler;
- [x] stable field names and resource attributes;
- [x] trace/span context enrichment;
- [x] tests for redaction and correlation fields;
- [x] removal of duplicated logging setup where practical.

Implementation prompt:
[Phase 1 ChatGPT prompt](prompts/logging-phase-1-shared-foundation.md)

### Phase 2 - Loki, collector pipeline, Compose and Helm

Add Loki in single-binary mode, store its durable TSDB index and chunks in the
dedicated SeaweedFS S3 bucket `myota-loki`, and connect the existing OTel logs
pipeline to Loki through OTLP. Keep Compose and Helm behavior aligned. Reuse the
SeaweedFS endpoint, bucket provisioning, and S3 credential/Secret patterns
already used by MyOTA.

Deliverables:

- [x] Loki TSDB and SeaweedFS S3 configuration for bucket `myota-loki`;
- [x] Compose observability service and a small local writable working volume;
- [x] Helm Loki workload/service and a small local writable working volume, without
  a 10-20 GiB Loki data PV;
- [x] local working storage limited to active TSDB/WAL/index/cache and compactor
  working data; object storage is the durable store;
- [x] OTel Collector Loki exporter path;
- [x] Grafana Loki datasource provisioning;
- [x] configurable 14-day Loki retention, enforced by the Loki compactor;
- [x] no separate MyOTA or SeaweedFS cleanup worker or object-store lifecycle policy
  for Loki retention;
- [x] Focused OTLP-to-Loki smoke check and SeaweedFS configuration check.
- [ ] Restart, retention-expiry, and full Compose/Helm parity acceptance checks.

Implementation prompt:
[Phase 2 ChatGPT prompt](prompts/logging-phase-2-loki-platform.md)

### Phase 3 - HTTP request logging and distributed trace propagation

Instrument the common HTTP path with completion logs for failed requests only
(4xx and 5xx), alongside proper W3C trace-context extraction/injection. Successful
requests remain visible in metrics and traces without a completion log.

Deliverables:

- [x] `http.request.completed` events;
- [x] normalized route, method, status and duration;
- [x] request/correlation IDs in logs and responses;
- [x] W3C `traceparent` / `tracestate` handling;
- [x] consistent server spans rather than isolated manual spans;
- [x] error classification without leaking request bodies;
- [x] tests confirming no completion log for 2xx, logs for 4xx/5xx, and
  propagated trace context;
- [x] live Identity `/healthz` request produced a matching Loki log record and Tempo trace.

Implementation prompt:
[Phase 3 ChatGPT prompt](prompts/logging-phase-3-http-tracing.md)

### Phase 4 - Workers, NATS/JetStream and asynchronous correlation

Add structured lifecycle logs to asynchronous components and propagate
correlation/trace metadata through messages.

Target components include:

- `activity-worker`;
- `activity-notifications`;
- `activity-adif-retention`;
- `geodata-import-processing`;
- `geodata-import-retention`;
- `identity-maintenance`;
- `outbox-core`;
- `outbox-activity`;
- `outbox-geo`.

Standard worker events:

```text
job.received
job.started
job.completed
job.retry
job.failed
```

Standard messaging fields should include `messaging.system=nats`, destination,
message/event ID, operation, and delivery attempt where available.

Deliverables:
- [x] lifecycle logs for Activity work and notification consumers;
- [x] lifecycle logs for Geodata import processing;
- [x] lifecycle logs for Identity maintenance and Deploy outbox relay;
- [x] NATS consumer/producer spans and W3C header propagation helpers, with
  focused helper tests;
- [x] Activity ADIF retention lifecycle logs;
- [x] Geodata import-retention source-owner coverage and validation: the
  authoritative worker and retention implementation are in
  `myota-geodata-service/geodata_retention_worker.py` and
  `myota-geodata-service/import_retention.py`; focused tests and the scheduled
  deployed job succeeded.
- [ ] deployed-worker retry/ACK and request-to-worker correlation acceptance;
  the initiating HTTP correlation ID is still not persisted by every outbox
  producer.
- [x] Stable NATS correlation header source for work/fact events: the relay
  selects `correlationId`, then `workId`, then `eventId`, and consumer log
  context inherits `Myota-Correlation-Id`. Focused CI tests pass; live
  request-to-worker acceptance remains open.

Implementation prompt:
[Phase 4 ChatGPT prompt](prompts/logging-phase-4-workers-nats.md)

### Phase 5 - Grafana log exploration and metrics/traces/logs correlation

Turn Loki into a first-class Grafana datasource and connect all three observability
signals.

Deliverables:

- [x] Loki datasource in Compose and Helm provisioning;
- [x] Loki derived field linking `trace_id` to Tempo;
- [x] Tempo trace-to-logs configuration;
- [x] service/environment/severity/component filters;
- [x] a MyOTA Logs / Failures dashboard;
- [x] panels for errors by service, recent failures, worker failures/retries and HTTP
  server-error logs;
- [ ] links from existing operational dashboards where useful;
- [x] verify Helm ConfigMap mounting and Grafana provisioning of the dashboard;
- [x] execute all seven mounted dashboard LogQL queries against live Loki OTLP
  structured metadata (no JSON parser);
- [ ] verify Grafana UI trace/log navigation in both directions; revision 225
  mounted the fix, but the authenticated UI links were not exercised.

Implementation prompt:
[Phase 5 ChatGPT prompt](prompts/logging-phase-5-grafana-correlation.md)

### Phase 6 - Production hardening, retention, alerts and verification

Finalize privacy controls, bounded cardinality, retention, runbooks and a small set
of log-specific alerts.

Use Prometheus for conditions already represented as metrics. Loki alerts should be
reserved for discrete exceptional events such as:

- database migration failure;
- permanent object-retention failure;
- JetStream maximum-delivery exhaustion;
- award rendering failure;
- unexpected authentication/signing failure.

Deliverables:

- [x] production 14-day Loki compactor retention configuration;
- [x] storage sizing/measurement procedure in the [runbook](logging-runbook.md);
- [x] Initial ingestion and object-store size measured; conservative capacity
  thresholds are documented with sample duration and limitations. The short
  pre-live sample is not represented as a production growth rate.
- [x] live Loki label-name check found only `deployment_environment`,
  `service_name`, and `service_namespace`;
- [ ] cardinality under representative production traffic and payload-label audit;
- [x] source redaction/correlation tests pass in service CI;
- [ ] live redaction check against Collector-to-Loki records;
- [ ] exceptional-event alert rules;
- [x] initial operational queries and troubleshooting runbook in
  [logging-runbook.md](logging-runbook.md);
- [ ] live ingestion, capacity, retention, and end-to-end verification;
- [ ] Compose/Helm parity check;
- [ ] end-to-end test: request -> trace -> log -> async worker correlation.

Implementation prompt:
[Phase 6 ChatGPT prompt](prompts/logging-phase-6-production-hardening.md)

### Phase 7 - PostgreSQL and PostGIS database observability

Add bounded database health and workload telemetry through the existing
OpenTelemetry and Prometheus stack. Direct Collector PostgreSQL receivers were
tested and failed authentication with the same Secret that worked for read-only
psql. The accepted implementation uses the Operations service as a bounded,
read-only adapter. Do not add query sampling or statement logging.

Deliverables:

- [x] Read-only monitor roles provisioned for core, Activity, and Geo; the
  Operations sampler uses fixed database names and read-only transactions.
- [x] PostgreSQL availability, connection use, database size, commit/rollback,
  blocks-read, and deadlock metrics are active in Prometheus for all three
  databases.
- [x] Psycopg client spans use pinned instrumentation with parameter capture
  and SQL commenter disabled; the Collector removes SQL-bearing attributes
  before Tempo export. A live read-only span check found no SQL statement,
  query text, or parameter attributes.
- [x] Geo PostGIS extension presence and bounded version are reported. Existing
  Geodata application instrumentation remains the owner of query-performance
  measurements; no database-wide spatial index scans are added.
- [x] The PostgreSQL/PostGIS dashboard reads the live Operations metrics and is
  provisioned in Compose and Helm.
- [x] Decision: do not collect raw PostgreSQL server statement logs. They can
  include SQL literals and user data. Aggregate DB counters plus sanitized
  application and Operations error logs provide the required operational
  signal without adding a second sensitive log path.
- [ ] Full Compose runtime parity remains open because Docker/Compose is not
  installed on the host. YAML parsing, Helm validation, Collector validation,
  and the live Helm rollout passed.

Do not expose SQL literals, bind parameters, user data, or complete geometry
payloads. Do not duplicate database measurements already owned by applications.

Implementation prompt:
[Phase 7 ChatGPT prompt](prompts/logging-phase-7-database-observability.md)

## Phase dependencies

```mermaid
flowchart LR
  P1[Phase 1
Shared foundation] --> P3[Phase 3
HTTP + trace context]
  P1 --> P4[Phase 4
Workers + NATS]
  P2[Phase 2
Loki platform] --> P5[Phase 5
Grafana correlation]
  P3 --> P5
  P4 --> P5
  P3 --> P7[Phase 7
PostgreSQL + PostGIS]
  P2 --> P7
  P1 --> P7
  P5 --> P6[Phase 6
Production hardening]
  P7 --> P6
```

Phases 1 and 2 may proceed in parallel. Phase 3 and Phase 4 depend on the common
semantics from Phase 1. Phase 5 requires Loki plus instrumented data. Phase 7
requires the Collector path from Phase 2 and database client instrumentation
from Phase 3. Phase 6 is the final production gate after Phases 5 and 7.

## Definition of done

Logging is considered implemented when:

1. Every production HTTP service and asynchronous worker emits structured logs.
2. Production logs reach Loki through the OTel Collector.
3. Compose and Helm expose equivalent logging capability.
4. Grafana can navigate from a log to its Tempo trace and from a trace to related logs.
5. A `correlation_id` can reconstruct a multi-service asynchronous workflow.
6. High-cardinality identifiers are queryable fields but not Loki labels.
7. Secrets and configured sensitive payload classes are demonstrably excluded.
8. Loki storage/retention is bounded and documented.
9. Existing Prometheus alert ownership is not duplicated by unnecessary log alerts.
10. Operational documentation includes troubleshooting queries and failure-mode guidance.
11. PostgreSQL health/workload metrics, database client traces, and safe database
    logs are visible through the existing OpenTelemetry/Grafana stack.
12. PostGIS-specific signals are covered directly where supported; otherwise the
    operations service exposes them through the existing telemetry path.
