# SeaweedFS Admin UI status and Grafana editing

Implemented in the operations service, Admin UI, contracts and Compose/Helm.
Select **Platform health → SeaweedFS storage** (`/object-storage`). The page
refreshes visible data every 30 seconds, offers manual refresh, and shows
paged sampled history with a detail modal. Its Grafana link opens the existing
MyOTA Object Storage dashboard.

## Data and ownership

The browser reads `GET /v1/operations/object-storage` and
`GET /v1/operations/object-storage/snapshots?page=1&pageSize=20`. The
operations service polls the private S3 `/status` and SeaweedFS `/metrics`
endpoints, records one sample per 30-second slot, and retains seven days in
its `operations_storage_snapshot` table in `myota_core`. Page size is capped
at 50. Connection concurrency is bounded to four per process. Native storage
metrics continue through the existing OpenTelemetry Collector to Prometheus;
operations also exports sample health and last-sample timestamps.

The service-owned migration `002_storage_snapshots.sql` is synchronized to
platform/deploy `db/migrations/core/003_storage_snapshots.sql`; the migration
runner discovers it automatically. The browser has no storage, database or
broker credentials and never reads those systems directly.

- Bucket object counts, logical bytes, physical bytes and read-only state
  are native exporter gauges. Only reported buckets are listed; empty or
  inactive buckets may be omitted. This page does not enumerate S3 objects.
  Gauge update cadence is controlled by SeaweedFS; capture time describes
  the probe, not proof of immediate synchronization after every object write.
- Filesystem total/used/available bytes come from volume-server resource
  gauges. With K3s local-path storage these describe a shared host filesystem,
  not the PVC quota or the size of MyOTA's objects alone.
- S3 counters are grouped by operation and status, cumulative since exporter
  startup and may reset. Grafana provides rates and latency distributions.
- Unknown gauges are null and displayed as **Unavailable**, never fabricated
  zeros. Failed probes are PARTIAL or UNAVAILABLE; old samples are marked
  stale after three configured poll intervals. Bucket truncation is explicit.
- Each HTTP probe times out after five seconds; metrics text is capped at
  2 MiB and listed buckets at 100 by default. Only sanitized status is saved;
  no object keys, contents, raw exporter responses or credentials are logged.

## Access and Grafana roles

Status/history requires GLOBAL_OPERATOR/GLOBAL_ADMIN, `observability.view`,
`operations.read`, or wildcard permission. Grafana uses a distinct stable
`myota:<account-id>` identity rather than the former shared Viewer identity.
`GET /v1/operations/observability-session` validates the access token and
checks `/v1/identity/me` for current session validity and role assignments.
GLOBAL_OPERATOR and GLOBAL_ADMIN map to Grafana **Editor** and can add
dashboards/panels. Other authorized readers map to **Viewer**. Wildcard scopes
alone do not grant editing.

Nginx uses that endpoint as its auth subrequest, forwards only the trusted
response headers through the gateway, and replaces incoming Grafana identity
and role headers. Grafana role synchronization runs on every request;
independent login tokens and anonymous access remain disabled. No public
Grafana ingress is needed. Access through `/observability/` still uses the
path-scoped MyOTA session cookie and checks every request.

All five provisioned dashboards start at **last 30 minutes** and refresh every
**30 seconds**. The file provider permits Editor UI saves. Independently
created dashboards persist in Grafana's database/PVC. Provisioned dashboard
UI edits can be overwritten by subsequent source provisioning updates; use
**Save as copy** for a dashboard maintained independently of Git. See
[Grafana auth proxy](https://grafana.com/docs/grafana/latest/setup-grafana/configure-access/configure-authentication/auth-proxy/)
and [dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).

## Configuration and verification

Compose supplies the private endpoints and Identity URL. Helm values
`services.operations.storageHealthUrl` and `storageMetricsUrl` default to
the configured SeaweedFS S3 health endpoint and the private metrics listener;
`services.identity.internalUrl` defaults to `http://myota-identity:8001`.
Set the storage URLs for an external SeaweedFS installation. Runtime variables
are `OPERATIONS_STORAGE_HEALTH_URL`, `OPERATIONS_STORAGE_METRICS_URL`,
`OPERATIONS_MAX_BUCKETS`, `OPERATIONS_POLL_SECONDS`, `OPERATIONS_HISTORY_DAYS`,
and `MYOTA_IDENTITY_URL`. No new credential Secret is required.

Verification must check authorized latest/history responses, missing-data and
failure semantics, actual Grafana Editor access, creation/deletion of a test
dashboard, Viewer denial, all dashboard time/refresh defaults, migration
success, and Helm/Fleet readiness. Deployment evidence is recorded in
[changes.md](changes.md) and the [9 October delivery evidence](evidence/observability-2026-10-09.md).
Existing Identity/Programme scrape coverage and the
broader structured-logging roadmap remain separate open work.
