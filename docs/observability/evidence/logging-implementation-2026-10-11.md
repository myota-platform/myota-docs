# Structured logging implementation evidence — 2026-10-11

## Overall status

**In progress; Phases 1–5 have source/configuration work on main, but their
production acceptance gates are not all closed. Phase 6 remains open. Phase 7
has monitor-role provisioning and a database dashboard asset, while its active
PostgreSQL receiver is paused after authentication failures.** No claim of
implementation completion is made.

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
| 5 — Grafana correlation | All seven queries from the mounted dashboard returned successfully against live Loki structured metadata. Helm ConfigMap/mount, Grafana dashboard provisioning, and the label-type `trace_id` derived-field config were observed in revision 225. | Authenticated Grafana log-to-trace and trace-to-log clicks remain unverified; links from other operational dashboards remain open. |
| 7 — PostgreSQL/PostGIS | Dedicated non-superuser monitor roles and external secret references exist. Read-only smoke jobs queried `pg_stat_database` in core, Activity, and Geo. The six-panel PostgreSQL/PostGIS dashboard JSON is mounted and parses in Grafana. | The Collector PostgreSQL receivers failed authentication despite successful `psql` checks with the same Secrets. They have been removed from the active pipeline. Client spans, safe DB logs, PostGIS direct/fallback signals, dashboard data, and Compose parity remain open. |

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
  its synchronized mirror. Helm revision 223 deployed the service image digests; revision 225 mounted the
  corrected Grafana assets. All application, worker, Loki, Collector, Grafana,
  and Tempo workloads were Ready. Grafana logged dashboard provisioning, while
  Fleet is Ready=True at Deploy main commit `904bef2a874e480d30b14ba2c6371b817b387d02`.
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
- All seven LogQL queries from the dashboard mounted in the Grafana pod were
  executed against live Loki. All returned HTTP/API success; the warning/error,
  retry, and 5xx panels returned zero matching results in the sampled window,
  not query errors. The prior parser generated `JSONParserErr` because the OTLP
  body is an event string while attributes are separate structured metadata.
  Deploy and Platform now contain direct structured-metadata filters and a
  label-type `trace_id` derived field; Helm revision 225 mounted the corrected
  config and Grafana logged dashboard provisioning. The actual authenticated UI
  links in both directions have not yet been clicked/verified.
- The dashboard ConfigMap, mount, and Grafana provisioning logs were checked
  after Helm revision 225. The service images and Geodata digest were already
  deployed by revision 223. Loki and Collector were Ready during the live
  query checks.
- The temporary disposable-broker smoke test remains cleaned up. No production
  business data was injected.

## Phase 6 — production hardening

Phase 6 is **not complete**. The following decisions apply until the live
acceptance gates are run:

- Initial Loki counter sample at 2026-10-11 02:37:48 UTC: since Loki pod start
  at 01:25:13 UTC, distributor counters recorded 1,759,104 OTLP bytes, including
  1,710,086 structured-metadata bytes and 2,185 log lines. This is a single,
  roughly 73-minute post-restart sample, not a representative daily rate or a
  capacity threshold. SeaweedFS bucket growth has not yet been measured.

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

Phase 7 is **in progress and not complete**. Deployment-owned assets have been
added, but PostgreSQL receiver metrics are not currently collected.

### Decisions and source changes

- Use separate non-superuser `myota_observability` logins and externally
  provisioned passwords. Explicit grants are limited to database connection and
  `SELECT` on `pg_catalog.pg_stat_database`. Do not grant `pg_monitor`,
  use application/superuser credentials, or enable query sample/top-query
  collection.
- Compose role bootstrap is in
  `myota-deploy/compose.yaml`. Helm role provisioning and Collector Secret
  references are in `myota-deploy/deploy/helm/myota/templates/observability.yaml`.
  The monitor passwords are not committed.
- PostgreSQL receiver configuration is in
  `myota-deploy/observability/otel-collector.yaml` and
  `myota-deploy/deploy/helm/myota/observability/otel-collector.yaml`.
  The same configuration is mirrored by `myota-platform`. The database
  receiver blocks were removed from the active metrics pipeline after live
  authentication failures; do not count this as metric delivery.
- The six-panel dashboard is
  `myota-deploy/observability/grafana/dashboards/myota-postgresql-postgis.json`
  and the Helm asset under
  `myota-deploy/deploy/helm/myota/observability/grafana/dashboards/`.
  Matching assets exist in the Platform mirror.

### Live checks on 11 October 2026

- The PostgreSQL monitor role hook completed successfully in Helm revision 229.
  A temporary read-only job connected to all three services and queried
  `pg_stat_database`; a separate check queried the receiver discovery
  database. No business rows were inserted or changed.
- The effective privilege inspection found no Core or Activity application
  relations readable by the monitor role. In Geo, the readable non-system
  relations were PostGIS-owned public metadata/reference objects:
  `public.geography_columns`, `public.geometry_columns`,
  `public.spatial_ref_sys`, `tiger.*`, and `topology.*`. These are visible
  through extension/default public grants; no explicit application-table grant
  was made. Confirm application-owner ACLs as part of the remaining review.
- The pinned contrib 0.135.0 receiver failed with PostgreSQL password
  authentication errors in all three databases, including in a temporary
  diagnostic Collector using the same ConfigMap and Kubernetes Secrets. The
  temporary `psql` job using those same Secrets succeeded. The discrepancy
  remains unexplained; credentials were not printed.
- To stop repeated failed scrapes in the live Collector, Deploy commit
  [`c7da003`](https://github.com/myota-platform/myota-deploy/commit/c7da00343134992a019b35f923287741bfa3e339)
  removes the PostgreSQL receivers from its active metrics pipeline. The
  matching Platform mirror commit is
  [`a4336a9`](https://github.com/myota-platform/myota-platform/commit/a4336a9b3b6f8d24989920bc7aa432b9f2bf0eed).
- The mounted Grafana dashboard JSON parsed successfully and contains six
  panels. Database receiver query data has not been accepted or verified. The
  existing Geodata application-owned PostGIS query histogram remains a separate
  available signal.
- After receiver removal, the Collector Deployment was Ready 1/1 and its recent
  log window had no PostgreSQL receiver errors. All Deployments were Ready 1/1.
  Helm revision 232 is deployed and is the current Fleet-applied release
  (`Ready=True`, sync generation 93). The current source bundle reports
  67/67 resources ready and 0 not ready. All Deployments meet their requested
  replica counts.
- Temporary diagnostic Jobs were deleted after the checks. The ephemeral
  debug pod was deleted and its Deployment replacement became Ready. No
  database test data was created.

The least-privilege decision follows the pinned
[Collector contrib v0.135.0 PostgreSQL receiver documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.135.0/receiver/postgresqlreceiver/README.md),
which requires `SELECT` on `pg_stat_database` for monitoring and reserves
`pg_monitor` for optional query sampling. Query sample/top-query stay disabled
because query text and plans can reveal sensitive information.

### Remaining Phase 7 work

- Resolve the receiver-versus-`psql` authentication discrepancy with a
  reproducible isolated test, then enable the receiver only after live metrics
  appear for all three databases without scrape errors.
- Audit effective ACLs against application-owned relations and extension
  defaults.
- Add privacy-safe database client spans and qualify trace attributes.
- Add safe PostgreSQL server logs, or document an accepted limitation and
  owner.
- Add supported bounded PostGIS health/workload signals; use Operations only
  for signals that have no safe direct integration.
- Execute and document Grafana queries against real metrics, retention,
  redaction, cardinality and resource behavior.
- Validate the full database telemetry path in Compose and Helm; do not treat
  the prepared assets as parity evidence.


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
