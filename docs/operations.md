# Local and production operations

## Local

1. Start Colima with Kubernetes enabled: `make colima-start`.
2. Run dependency-free tests: `make test`.
3. For the browser slice: `make run`.
4. For the durable local stack: `make compose-up`.
5. For Kubernetes: `make k8s-install`, then `kubectl -n myota get pods` and `kubectl -n myota port-forward svc/myota-gateway 8080:8080`.

The Compose stack is the supported local runtime for persisted data. It runs
three database containers: plain PostgreSQL for `myota_core` and
`myota_activity`, and PostGIS for `myota_geo`. It supplies
`CORE_DATABASE_URL` to identity/programme/control-plane processes,
`ACTIVITY_DATABASE_URL` to activity, award, worker and activity-outbox
processes, and `GEO_DATABASE_URL` to geodata/import processes. Every
database-backed workload receives
`MYOTA_REQUIRE_DURABILITY=1`. A service
with a missing database URL exits during startup rather than using process
memory. The named core, activity and geodata volumes survive container
restarts; do not use the
dependency-free `make run` process for data you need to keep.

`make test` remains dependency-free by design: unit tests explicitly exercise
the in-memory adapter and do not represent the production or Compose storage
path. Binary imports and award assets are stored in the mounted SeaweedFS volume;
their metadata, queues, audit events, QSO data and award state are persisted in
PostgreSQL.

Large browser uploads are spooled to the geodata upload-spool volume and the
HTTP endpoint returns `202 UPLOAD_PENDING` after the request body has been
received and scanned. The background handoff stores the source in SeaweedFS
and then queues preprocessing. `UPLOAD_PENDING` runs and their spool files are
recovered after a geodata restart; the browser should follow the import run
status rather than wait for SeaweedFS storage to finish.

### Import recovery

Import history is durable and should be used as the operational source for
file visibility and processing status. A queued or processing import has its
source document in SeaweedFS and an execution lease in the geodata
`import_run` table. Heartbeats keep active work leased. When the geodata
service starts, queued runs and runs left in `PROCESSING` by the previous
instance are requeued immediately and persisted before recovery workers are
dispatched; this does not wait for the normal lease timeout. A recovered run
increments `attempt_count` when it claims the work and continues from the
original source. Pasted KML/GPX uses its normalized GeoJSON recovery snapshot.
Binary formats without an installed parser remain visibly queued. Runs with no
recoverable source are changed to `FAILED` with `last_error`, so they do
not appear indefinitely as active work.

### Coordinate reference systems

The geodata intake boundary stores validated geometry in WGS84 longitude and
latitude (`EPSG:4326`). GeoJSON and ArcGIS `spatialReference` declarations are
read before validation and supported source CRSs are reprojected with `pyproj`.
Shapefile ZIP uploads use a matching `.prj` sidecar. This allows projected
datasets such as Spain's ETRS89 / UTM 30N (`EPSG:25830`) to be imported without
manual coordinate conversion. The original declaration is retained in
`provenance.sourceCrs`; datasets without a declaration retain the legacy
WGS84 assumption.

For diagnosis, inspect `status`, `attempt_count`, `heartbeat_at`,
`lease_until`, `last_error`, `filename` and `stats` in `import_run`, then check
the corresponding SeaweedFS object under the recorded bucket and object key.
Do not delete the PostGIS or SeaweedFS volumes while investigating an import.

### Client migration and operational dashboard

Phase 4 moves first-party lifecycle writes to the preferred resource APIs. The
contract clients and migration rules are recorded in
[`api-phase4-client-operational-migration.md`](api-phase4-client-operational-migration.md).
All HTTP services expose `/metrics`; OpenTelemetry adds request traces and
request duration metrics, and the OpenTelemetry Collector is the single
collection boundary for Prometheus and Tempo. Domain gauges are read from
durable identity, programme, PostGIS, and activity state. They include users,
callsigns, programmes, categories, entities, imports, activations, QSOs,
participants, awards, jobs, corrections and queue lag. Start the optional
local dashboard with:

```bash
docker compose --profile observability up -d prometheus grafana
```

The collector is started automatically by the `observability` profile. Grafana
is available at `http://localhost:3000`, Prometheus at
`http://localhost:9090`, the collector's Prometheus exporter at
`http://localhost:8889`, and Tempo at `http://localhost:3200`. See
[`observability.md`](observability.md) for the trust model and interpretation
of empty or unavailable series.

Use `make verify-phase4` after rebuilding the durable stack. It runs the
authorization, idempotency, audit, activity, award, and geodata regression
suite, verifies typed-client coverage, and probes gateway/service health and
metrics endpoints. Keep compatibility aliases enabled until the dashboard
shows no remaining callers and the Phase 5 review approves their removal.

### Database restart recovery and large staged imports

The local Compose stack uses `restart: unless-stopped` for the database,
service, gateway, worker, and outbox containers. This is important because a
PostgreSQL restart temporarily rejects connections; workers must be restarted
automatically instead of exiting permanently on the first connection error.
Kubernetes Deployments provide the equivalent restart behavior.

Large GeoJSON imports can create substantial staged candidate data because the
candidate geometry and provenance remain reviewable before promotion. Local
Compose therefore runs one geodata import worker at a time to bound concurrent
memory use. A large run should be allowed to finish preprocessing before
validation or finalization; do not remove the database or object-store volumes
to recover from a transient outage.

Geodata entity lifecycle, geometry, and category edits are persisted in the
relational PostGIS tables. The geodata service's JSON `service_state` row is a
compatibility snapshot only; on startup, PostgreSQL entity columns take
precedence over an older snapshot. The geodata service has no built-in entity
seed data; populate a local catalogue through the import or community-proposal
workflow instead.

## Production notes

Use managed PostgreSQL where possible, enable PostGIS, store credentials in Kubernetes Secrets or an external secret manager, and back up core and geodata databases independently. Pin image digests, enforce network policies so services reach only their own database, and expose QGIS access only through a private network or bastion.

### Rancher Fleet on K3s

The Fleet bundle and Traefik ingress configuration are maintained in
[`myota-deploy/deploy/helm/myota`](https://github.com/myota-platform/myota-deploy/tree/main/deploy/helm/myota).
The [Spainip deployment guide](https://github.com/myota-platform/myota-deploy/blob/main/deploy/helm/myota/DEPLOYMENT.md)
covers DNS/TLS, the three database StatefulSets or external database endpoints,
required Kubernetes Secrets, Fleet GitRepo setup, persistent volumes, rollout
checks, and current chart limits. The chart-managed option provides three
separate persistent targets, with PostGIS only on `myota_geo`; it is
single-instance and requires off-host backups. Keep credential values out of
Fleet values files and Git.

The chart supports separate Traefik hostnames: `ingress.host` (the API gateway,
default `api.myota.top`) routes to the API gateway, while
`ingress.adminHost` routes to the administration web. The admin web proxies
same-origin `/v1` requests to the gateway. The Spainip sample relies on the
existing Traefik installation to provision certificates automatically and
leaves TLS secret names empty; explicit secrets remain configurable for other
clusters. Defaults are `api.myota.top` for the API and `admin.myota.top` for
the admin UI; both can be changed independently in Helm values. The `myota-web`
participant client is not yet packaged as a container, so the public hostname
currently exposes the API rather than a participant website. Helm lint/render
runs in GitHub Actions; Fleet performs deployment from the reviewed Git
revision.

### Container image publishing

GitHub Actions builds and publishes chart images to GHCR on pushes to `main`;
each owning repository also exposes `workflow_dispatch` for an immediate
build. These workflows currently publish the mutable `latest` tag, so pin
immutable tags before treating a rollout as a reproducible production release.
Image ownership is aligned with the Helm values:

| GHCR image | Build repository |
| --- | --- |
| `ghcr.io/myota-platform/myota-service` and `myota-gateway` | `myota-deploy` |
| `ghcr.io/myota-platform/myota-admin-web` | `myota-admin-web` |
| `ghcr.io/myota-platform/myota-identity-service` | `myota-identity-service` |
| `ghcr.io/myota-platform/myota-programme-service` | `myota-programme-service` |
| `ghcr.io/myota-platform/myota-geodata-service` | `myota-geodata-service` |
| `ghcr.io/myota-platform/myota-activity-service` | `myota-activity-service` |

The participant `myota-web` is not yet in the Helm release and has no
container image workflow. The workflows require the repository Actions token
to have GHCR package write permission.

## Activity capacity controls

The activity API is stateless and can be scaled horizontally. Each pod has a
bounded HTTP worker ceiling and a bounded PostgreSQL pool; do not increase
both without checking PostgreSQL connection limits. Helm defaults are three API
replicas, a pool of twelve per API pod, two activity workers, and a separate
notification consumer.

The QSO table is normalized and indexed for programme, participant, callsign,
entity, activation date and QSO timestamp. Use the COPY batch path for bulk
loads. Introduce date partitioning only after measured table size/query plans
justify it; it is a lifecycle and maintenance tool as much as a performance
tool.

Before production, load-test sustained and burst QSO ingestion, duplicate
retries, ADIF import throughput, map/public-history reads, award progress,
leaderboards and PDF rendering. Record p95/p99 latency, PostgreSQL CPU/IO,
connection usage, lock waits, job lag, outbox lag and object-store failures.
