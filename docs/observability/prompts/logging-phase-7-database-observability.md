# Phase 7 prompt - PostgreSQL and PostGIS observability

Use this prompt in ChatGPT Work or Codex to implement Phase 7 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Add PostgreSQL and PostGIS database health, workload and diagnostic telemetry to
MyOTA's existing OpenTelemetry and Grafana observability stack.

Repositories to inspect:
- myota-platform/myota-deploy
- myota-platform/myota-operations-service
- myota-platform/myota-docs
- service repositories that own PostgreSQL/PostGIS connections and queries

Inspect the deployed PostgreSQL/PostGIS topology and versions, Collector
distribution and pinned version, Compose and Helm configuration, database
credential patterns, service database instrumentation, existing metrics,
dashboards and runbooks before changing anything.

Prefer direct OpenTelemetry integrations:

1. Use the OpenTelemetry Collector PostgreSQL receiver for server/database
   metrics when it is compatible with the pinned Collector distribution and
   PostgreSQL version. Verify its maturity, metric coverage, query overhead and
   required grants. Use a dedicated least-privilege monitoring role and existing
   secret/TLS patterns.
2. Instrument application-side PostgreSQL client calls with supported
   OpenTelemetry database instrumentation. Use database semantic conventions,
   preserve trace context, and emit low-cardinality summaries and duration/error
   information. Never expose SQL literals, bind values, credentials or sensitive
   query text.
3. Collect PostgreSQL server logs through a supported Collector file/OTLP
   pipeline when practical. Preserve rotation/offset behavior and apply
   redaction before export. Do not log statement text or parameters by default.
4. Determine which PostGIS-specific signals are not covered by the PostgreSQL
   receiver or application instrumentation. Use direct OTel SQL/query collection
   only when its stability, overhead, least-privilege requirements and cardinality
   are acceptable. Suitable bounded signals can include PostGIS extension
   presence/version, spatial index availability/use, and spatial-query duration
   or error counts. Avoid frequent full-catalog scans and geometry/payload export.
5. If a PostgreSQL or PostGIS metric, trace, or log cannot be collected directly
   and safely through a supported integration, extend
   `myota-platform/myota-operations-service` to expose that missing signal via
   its existing OTel/metrics/logging path. Keep the operations service as a
   read-only observability adapter; do not create a second telemetry backend or
   duplicate signals already collected directly. Document why the fallback is
   needed and its ownership.

Implementation requirements:

- Keep metrics, traces and logs in the existing OpenTelemetry Collector,
  Prometheus, Tempo and Loki architecture.
- Keep Compose and Helm behavior aligned.
- Reuse existing PostgreSQL credential, Secret, network and deployment patterns.
  Do not publish database or telemetry endpoints publicly.
- Keep metric labels bounded. Never label by SQL text, query parameters,
  geometry, user, entity, QSO, callsign, request or trace ID.
- Redact connection strings, credentials, SQL literals, bind values, personal
  data and complete spatial payloads from logs and spans.
- Keep Prometheus/application metrics as the alert source of truth when they
  already represent a condition. Do not add duplicate database alerts.
- Add useful Grafana database/PostGIS panels and links to related traces/logs.
- Update retention, security, deployment and troubleshooting documentation.
- Follow the roadmap's Loki SeaweedFS storage and compactor-owned retention
  requirements for any collected database logs.

Do not add new database extensions or privileges beyond what the selected
integration needs without documenting the reason. Do not use an alpha/development
Collector component in production without an explicit maturity and support
assessment and a documented fallback.

## Verification and completion evidence

Run the relevant Collector configuration checks, Compose profile checks, Helm
lint/render checks and service tests. Demonstrate:

- PostgreSQL metrics are present and low-cardinality;
- a representative application database operation has a trace linked to the
  originating request;
- PostgreSQL logs arrive with sensitive SQL/credentials removed;
- bounded PostGIS signals are present, either from a direct OTel integration or
  from the documented operations-service fallback;
- the monitoring role cannot modify application data;
- Compose and Helm expose equivalent capabilities;
- database telemetry collection remains healthy when the database is unavailable
  and reports unavailable rather than fabricated zero values.

Return changed repositories/files, exact verification evidence, required
configuration/secret names (never secret values), operational queries/runbooks,
and any unsupported signals or remaining risks.

## OpenTelemetry references

- [OpenTelemetry Collector PostgreSQL receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/postgresqlreceiver)
- [OpenTelemetry Collector SQL Query receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/sqlqueryreceiver)
- [OpenTelemetry Collector File Log receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver)
- [Database client span semantic conventions](https://opentelemetry.io/docs/specs/semconv/db/database-spans/)
- [PostgreSQL semantic conventions](https://opentelemetry.io/docs/specs/semconv/db/postgresql/)
