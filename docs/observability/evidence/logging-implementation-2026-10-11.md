# Structured logging implementation evidence — 2026-10-11

## Overall status

**In progress; Phases 1–5 have source/configuration work on main, but their
production acceptance gates are not all closed. Phases 6 and 7 remain open.**
No claim of implementation completion is made.

Authoritative application code is owned by the five service repositories:
Identity, Programme, Activity, Geodata, and Operations. Deployment behavior is
owned by `myota-deploy`; `myota-platform` is its synchronized mirror.
Documentation and phase evidence are owned by `myota-docs`.

## Phase gate summary

| Phase | Verified work | Open exit evidence |
|---|---|---|
| 1 — shared logging | Shared `myota_logging.py`, JSON logging, OTel logging provider/handler, resource attributes, redaction helpers, and focused unit tests exist in all five services. Deploy/Platform contain the shared runtime mirror. | Run the complete five-service CI/test evidence against the exact current commits and verify production images include the changes. |
| 2 — Loki platform | Deploy-owned Loki uses TSDB and SeaweedFS bucket `myota-loki`, 14-day compactor retention, a bounded working volume, and the existing Collector OTLP logs path. Grafana Loki datasource is provisioned. Loki and Collector are Ready in K3s. A disposable OTLP-to-Loki smoke test passed and its namespace was removed. | Compose was not started for parity/restart qualification. Long-term object-store growth and production retention behavior have not been observed. |
| 3 — HTTP tracing | Shared HTTP path emits completion events, normalizes routes, extracts/injects W3C trace context, and adds response trace IDs. Tests cover success, authorization failure, server error, and propagation. A live Identity health request now has a matching structured-metadata log and Tempo server span. | Exercise the Grafana link after its corrected datasource provision is deployed; qualify HTTP-to-worker correlation across services. |
| 4 — workers and NATS | Activity worker, notification consumer, Geodata worker, Identity maintenance, and Deploy outbox relay have structured lifecycle logs and messaging spans/headers. Sensitive payloads and exception text remain excluded. Helper propagation tests passed. | Worker retry/ACK integration and end-to-end async correlation have not been accepted against the live images. Some event/outbox envelopes do not carry a `correlation_id`; no synthetic business ID is added. Verify the Geodata Deploy mirror against its authoritative implementation. |
| 5 — Grafana correlation | Dashboard queries without `| json` and with `severity_text` returned successfully against live Loki structured metadata. Helm ConfigMap/mount and dashboard provisioning were observed. Deploy/Platform now provision a `trace_id` metadata-derived field. | The corrected datasource/dashboard revisions have not yet rolled out. Verify actual Grafana log-to-trace and trace-to-log navigation after Fleet applies them; useful links from other dashboards remain open. |

## Phase 1–5 implementation references

- Shared logging and tests: `myota-identity-service/myota_logging.py`,
  `myota-programme-service/myota_logging.py`,
  `myota-activity-service/myota_logging.py`,
  `myota-geodata-service/myota_logging.py`,
  `myota-operations-service/myota_logging.py`, and
  `tests/test_myota_logging.py` in each service.
- HTTP path: `common.py` and `otel.py` in the five service repositories;
  Deploy copies are `myota-deploy/services/common.py` and
  `myota-deploy/services/otel.py`.
- Worker instrumentation: `activity_worker.py`, `event_consumer.py`,
  `geodata_import_worker.py`, `identity_maintenance.py`,
  `services/outbox_worker.py`, and `services/geodata_retention_worker.py`.
  Activity ADIF retention is owned by
  `myota-activity-service/adif_retention.py`; Geodata import retention is
  owned by `myota-geodata-service/import_retention.py`.
- Loki/Collector and dashboard assets: `observability/loki.yaml`,
  `observability/otel-collector.yaml`,
  `observability/grafana/provisioning/datasources/datasources.yml`,
  `observability/grafana/dashboards/myota-logs-failures.json`, and their
  `deploy/helm/myota/` mirrors in Deploy and Platform.
- The dashboard uses Loki OTLP structured metadata rather than parsing its string event body as JSON. The live query run is recorded below.

## Host and CI evidence

- K3s namespace: `myota`; Deploy is authoritative for Helm and Platform is
  its synchronized mirror. At the latest check, Helm revision 223 was deployed;
  all application, worker, Loki, Collector, Grafana, and Tempo workloads were
  Ready. Fleet had applied Deploy commit
  `0bd491fa5ffd92a0a46c9aaf0978fad88bd4d872`.
- Running service digests:

| Owner | Running image digest | Build evidence |
|---|---|---|
| Identity | `sha256:d2feb9935e4d7e457a3cce6b39f30a81eb6ee00f26ef52744fd6a0575f2197b2` | [38104742283](https://github.com/myota-platform/myota-identity-service/actions/runs/38104742283), quality [38104743122](https://github.com/myota-platform/myota-identity-service/actions/runs/38104743122) |
| Programme | `sha256:d5a4b18098b98c9e9b6c4acac085b20c663c2361c9e9019f1d3b6de5ff38a145` | [38104741260](https://github.com/myota-platform/myota-programme-service/actions/runs/38104741260), quality [38104741752](https://github.com/myota-platform/myota-programme-service/actions/runs/38104741752) |
| Activity | `sha256:f88145de15a580d426d3452c12e6f652a16f6b433dd9441ddc7b7ac8d80654ce` | [38104743745](https://github.com/myota-platform/myota-activity-service/actions/runs/38104743745), quality [38104744128](https://github.com/myota-platform/myota-activity-service/actions/runs/38104744128) |
| Geodata | `sha256:f924eed50a6e8cc83b5399836cf597d293a0f95b3fb2e830cff78128d44a6311` | [38104897949](https://github.com/myota-platform/myota-geodata-service/actions/runs/38104897949), quality [38104898359](https://github.com/myota-platform/myota-geodata-service/actions/runs/38104898359) |
| Operations | `sha256:8a948db53794baf8a3e38125bc943182c9ac1f497863ab9525df0532444e2f0c` | [38103329678](https://github.com/myota-platform/myota-operations-service/actions/runs/38103329678) |
| Deploy workers | `sha256:8bb123325b4e1f0fc9d0c42947c7fc8cee91c8354ce54323266fb6d457491982` | [38105043371](https://github.com/myota-platform/myota-deploy/actions/runs/38105043371) quality; [38105042942](https://github.com/myota-platform/myota-deploy/actions/runs/38105042942) Helm validation |

- The four service entrypoints now call `configure_logging()`. Identity,
  Programme, Activity, and Geodata quality checks and image builds passed.
  Geodata's relational-boundary, load-harness, and upload-recovery jobs all
  passed in run 38104897949; its previously failing delivery timeout and
  formatter attempts were superseded by that successful run.
- A non-mutating Identity `GET /healthz` returned HTTP 200 and response
  `X-Trace-ID: 37e1a9421947ff6904ab225d5366ad0d`. Loki returned one matching
  `http.request.completed` record when filtering structured metadata directly
  by `trace_id`; the record also included `severity_text=INFO`, route, status,
  duration, request ID, and correlation ID. Tempo returned the matching trace
  with one `GET /healthz` span. The temporary port-forwards were stopped. No
  application/business rows or test data were created.
- Loki's `/loki/api/v1/labels` endpoint returned the three indexed label names
  `deployment_environment`, `service_name`, and `service_namespace`.
  The record's other OTLP attributes are queryable structured metadata. This is
  evidence about the current indexed-label set, not a representative cardinality
  or sensitive-field/redaction audit.
- The dashboard query shapes were executed against live Loki with the JSON
  parser removed and `severity_text` used. Log streams and metric aggregations
  returned success. The prior parser generated `JSONParserErr` because the
  OTLP body is an event string while attributes are separate structured
  metadata. Deploy and Platform now contain the correction: direct structured
  metadata filters and a label-type `trace_id` derived field. These commits
  have not yet been observed in a new Helm revision; UI trace/log navigation
  stays open until the corrected config is provisioned and exercised.
- Grafana logged successful dashboard provisioning from Helm revision 221; the
  dashboard ConfigMap and mount were also verified. Revision 223 subsequently
  rolled out the Geodata digest. Loki and Collector were Ready during the live
  query checks.
- The temporary disposable-broker smoke test remains cleaned up. No production
  business data was injected.

## Phase 6 — production hardening

Phase 6 is **not complete**. The following decisions apply until the live
acceptance gates are run:

- Keep Loki labels restricted to `service_name`, `service_namespace`,
  `deployment_environment`, `severity`, and `component`. Request, trace,
  correlation, domain IDs, URLs, and exception text remain fields rather than
  labels. No deployment evidence has yet demonstrated the final ingested label
  set under application traffic.
- Production DEBUG must stay disabled. Confirm this against each live workload
  environment and image; source defaults alone are insufficient.
- Keep Loki's 14-day compactor policy and SeaweedFS object bucket as configured.
  Do not add bucket lifecycle deletion or a separate cleanup worker. No actual
  retention expiry test or measured object-growth baseline exists yet.
- Do not add generic ERROR or availability alerts: Prometheus owns those
  conditions. No discrete exceptional-event Loki alert is enabled until the
  emitting event is verified in the deployed code and its no-payload/privacy
  behavior is tested.
- No capacity threshold is inferred from an empty/short-lived broker or an
  unmeasured log bucket. Establish ingestion bytes/day and bucket growth from a
  representative observation window before setting a scale-out threshold.
- Runbooks and end-to-end HTTP → trace/log → NATS → worker → trace/log
  qualification remain required. Current live images and missing source
  `correlation_id` fields prevent claiming a stable multi-service workflow.

## Phase 7 — PostgreSQL and PostGIS

Phase 7 is **not implemented or deployed**. No Collector PostgreSQL receiver,
monitoring login, server-log pipeline, database dashboard, or database tracing
acceptance is claimed.

Decision: use the pinned contrib Collector PostgreSQL receiver only with a
dedicated login granted `pg_monitor`, never the application or superuser
credential. Keep table/index/query-name metrics disabled unless a measured,
bounded need is approved. The receiver's metrics are beta in contrib 0.135.0,
so start with a small metric set and monitor receiver overhead. PostgreSQL
server logs must not be enabled with statement/parameter logging; capture only a
supported redacted log path if the collection and rotation behavior can be
qualified. Existing Geodata PostGIS query-duration metrics remain the direct
application-owned spatial workload signal. Use the Operations service only for
bounded PostGIS signals that lack a supported direct integration, preserving
its read-only adapter boundary.

The receiver assessment follows the pinned [Collector contrib v0.135.0
PostgreSQL receiver documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.135.0/receiver/postgresqlreceiver/documentation.md).

Required follow-up implementation and evidence:

- provision a separate monitoring credential through the host's secret process;
  never commit its value;
- apply an idempotent role grant to core, Activity, and PostGIS databases;
- configure equivalent Compose and Helm receivers with bounded collection
  intervals/metrics;
- add privacy-safe Psycopg database client spans without SQL text, values, or
  geometry;
- add supported redacted PostgreSQL log ingestion, or explicitly document why it
  remains unsupported;
- add PostGIS extension/index/workload views with a bounded query plan and
  interval;
- prove read-only permissions, collector outage/unavailable behavior, dashboard
  data, and Compose/Helm parity.

## Phase 6–7 exit criteria

Do not mark the overall plan complete until all items below have evidence:

1. The deployed application image digests contain the current logging changes.
2. Live HTTP, asynchronous worker, and NATS logs reach Loki with no prohibited
   labels or sensitive content.
3. A stable correlation ID reconstructs a real safe request-to-worker workflow,
   and log/trace links open both directions.
4. Loki's 14-day retention and SeaweedFS growth are measured; working-volume
   headroom is documented from observed data.
5. Only discrete log-specific alerts are enabled and their source events are
   proven; existing Prometheus alert ownership remains unchanged.
6. PostgreSQL metrics use a dedicated read-only monitoring role, database spans
   expose no SQL content, and safe server logs are visible or the unsupported
   signal is explicitly accepted with an owner.
7. Bounded PostGIS signals are visible with measured overhead.
8. Compose/Helm parity, Helm render/lint, Collector validation, service tests,
   runbooks, and dashboard queries have passed.
