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
| 3 — HTTP tracing | Shared HTTP path emits completion events, normalizes routes, extracts/injects W3C trace context, and adds response trace IDs. Tests cover success, authorization failure, server error, and propagation. Duplicate spans are disabled where existing OTel middleware owns the server span. | Verify the live services use the new image digests and inspect a real request trace/log pair in Loki and Tempo. |
| 4 — workers and NATS | Activity worker, notification consumer, Geodata worker, Identity maintenance, and Deploy outbox relay have structured lifecycle logs and messaging spans/headers. Sensitive payloads and exception text remain excluded. Helper propagation tests passed. | Worker retry/ACK integration and end-to-end async correlation have not been accepted against the live images. Some event/outbox envelopes do not carry a `correlation_id`; no synthetic business ID is added. Verify the Geodata Deploy mirror against its authoritative implementation. |
| 5 — Grafana correlation | Compose and Helm datasource provisions link Loki logs to Tempo and Tempo spans to Loki. The `MyOTA Logs / Failures` dashboard exists in Deploy/Platform Compose and Helm assets. Dashboard JSON parses. | Dashboard LogQL has not yet been exercised against live structured application logs. The Grafana pod has not yet been observed loading the latest main assets. |

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
- The latest dashboard fix inserted the required separation before LogQL range
  selectors. The JSON parser check passed; this does not substitute for Loki
  query execution.

## Host and CI evidence

- K3s namespace: `myota`. The last check found the Fleet bundle
  `myota-deploy-deploy-helm-myota` Ready=True.
- Loki `grafana/loki:3.7.8` and Collector
  `otel/opentelemetry-collector-contrib:0.135.0` were Ready.
- Fleet accepted Deploy main commit `c039d694206d45f7dc2ae4c4822feb6e6cf9768f`
  and returned Ready=True. New CI-published images rolled out and all desired replicas are Ready:

| Owner | Running image digest | Build evidence |
|---|---|---|
| Identity | `sha256:dc0b3482fc930e8abc4d4b8ff1db9dde802bc6c8edc15c72cd9d542af3c0c22b` | [38103324688](https://github.com/myota-platform/myota-identity-service/actions/runs/38103324688) |
| Programme | `sha256:d10a123a79f128bc226fef72ff3cbf891950294a07d354f21365b78082c8ce29` | [38103325704](https://github.com/myota-platform/myota-programme-service/actions/runs/38103325704) |
| Activity | `sha256:6760bb5a1db70cddec3e99e738bc38af19231090919f430e61861f4a006c161e` | [38103443803](https://github.com/myota-platform/myota-activity-service/actions/runs/38103443803) |
| Operations | `sha256:8a948db53794baf8a3e38125bc943182c9ac1f497863ab9525df0532444e2f0c` | [38103329678](https://github.com/myota-platform/myota-operations-service/actions/runs/38103329678) |
| Deploy workers | `sha256:8bb123325b4e1f0fc9d0c42947c7fc8cee91c8354ce54323266fb6d457491982` | [38103800310](https://github.com/myota-platform/myota-deploy/actions/runs/38103800310) |
| Geodata | `sha256:4c7c65a51a983f760176b6149ad389102a4a65b31cb4b2fc39a180008dfd86eb` | Still running the prior digest; latest image build [38103448580](https://github.com/myota-platform/myota-geodata-service/actions/runs/38103448580) failed and skipped publication. |

The logging changes are live in the five successful owners; Geodata has not
rolled out.
- A safe read-only Identity probe returned HTTP 200 from `/healthz` and HTTP 403
  from unauthenticated `GET /v1/identity/me`. The API pod emitted
  `http.request.completed` records. A Loki query over the matching test trace ID
  returned no records; a bounded count query showed Activity notifications and
  Operations logs only. Identity trace-to-log acceptance therefore failed and
  remains open. The temporary port-forwards were stopped; no database/business
  rows or other test data were created.
- Geodata's latest image publication workflow
  [38103448580](https://github.com/myota-platform/myota-geodata-service/actions/runs/38103448580)
  failed because its relational-boundary job reports seven JetStream worker
  delivery timeouts. Job output also shows the existing database error
  `Geodata writer is obsolete; deploy the row-authoritative service image`;
  the image build/publish job was skipped. Resolve this service test/deployment
  mismatch before expecting a new Geodata digest.
- Deploy Python quality passed on main after fixing formatting and two missing
  mirror imports: workflow run
  [38103761084](https://github.com/myota-platform/myota-deploy/actions/runs/38103761084).
  The preceding failures were fixed. The matching deployment image workflow
  [38103760646](https://github.com/myota-platform/myota-deploy/actions/runs/38103760646)
  completed successfully.
- Identity, Activity, and Geodata Python quality runs passed for their latest
  helper/worker commits (runs 38103325177, 38103444208, and 38103449017).
- The prior disposable Loki smoke test submitted one OTLP record and queried it
  successfully; its temporary namespace was removed. No production business
  data was inserted by this logging work.

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
