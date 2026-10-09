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
path. Binary imports, ADIF uploads, award assets and issued certificates are
stored in the mounted SeaweedFS volume; their metadata, queues, audit events,
QSO data and award state are persisted in PostgreSQL.

### Object-storage bucket boundaries

The S3 endpoint and credentials are shared, but distinct purposes use distinct
buckets:

| Bucket (configurable) | Owner / contents | Retention boundary |
|---|---|---|
| `myota-geodata-imports` | Geodata source files and import snapshots | Geodata cleanup removes eligible import objects and logs after 30 days, including stale, failed, pending and stalled runs. |
| `myota-adif` | Uploaded ADIF source logs | Completed and failed source objects are deleted 15 days after terminal processing; import results remain in PostgreSQL. Queued and processing imports are excluded. |
| `myota-award-assets` | Editable award background artwork | Retain while referenced by draft or published awards. |
| `myota-award-signatures` | Award-manager signature images | Retain while referenced by award issuance/template records. |
| `myota-certificates` | Generated issued certificate PDFs | Durable issuance artifacts; no short import-retention policy. |

The award API assigns background/signature buckets by asset kind; the
client-supplied `bucket` property is read-only and cannot select a destination.
Existing object references retain their recorded bucket. When upgrading from
the former shared `myota-awards` bucket, copy the objects and update asset
metadata before retiring that bucket. From the activity-service checkout, set
`ACTIVITY_DATABASE_URL` and the S3 endpoint/credentials, review a dry run, then
apply it:

```bash
python3 migrate_award_asset_buckets.py
python3 migrate_award_asset_buckets.py --apply
```

Run the migration in an environment that can reach both the durable activity
database and object store. It copies and verifies stored objects before
updating each asset's recorded bucket, safely re-points `MISSING` placeholders,
is safe to re-run, and intentionally leaves old source objects in place.
Verify no metadata still references `myota-awards` and take a backup before
retiring/deleting that legacy bucket. Never apply a blanket 30-day lifecycle
rule to all object buckets.

The activity service runs a separate daily ADIF source-retention job against
`activity_import`. It selects only terminal `COMPLETED` or `FAILED` imports
whose `completed_at` is at least 15 days old and whose source has not already
been removed. The job
deletes the object from the configured ADIF bucket, then sets
`source_deleted_at`; if object deletion or the marker update fails, a later
pass retries safely. The import row, result counts, diagnostics, and QSO
records remain in PostgreSQL. Queued and processing uploads are not eligible.
Award backgrounds, signatures, and issued certificates are never touched.
Configure Helm with `activityAdifRetention.enabled`, `schedule`,
`retentionDays`, and `batchSize`; local Compose uses the corresponding
`MYOTA_ADIF_RETENTION_*` settings and runs the worker daily.

Large browser uploads use `POST /v1/geodata/import-uploads` to create a
user-owned, 24-hour upload session, then send bounded binary parts to
`POST /v1/geodata/import-uploads/{uploadId}/parts/{partNumber}`. The browser
persists its session key and resumes by checking the received part list; a
retry of the same part number replaces that part, and each part SHA-256 is
verified. The default part size is 16 MiB (minimum 5 MiB except the last part),
and the maximum file size remains 1 GiB. `POST .../{uploadId}/complete` checks
contiguous parts, total size, object checksum, and malware gates before creating
the import run and durable outbox event. Source bytes live only in SeaweedFS;
the API uses bounded ephemeral scratch only for one part at a time. There is
no upload-spool PVC. A daily cleanup sweep aborts abandoned multipart sessions
after 24 hours and retains compact session history for 30 days. Keep any
SeaweedFS incomplete-multipart lifecycle rule aligned with this database
expiry.

### Import recovery

The browser supports pause/resume and discard of incomplete uploads, preserves
session state on transient errors and checks saved-part hashes against the
reselected original file. Each new submission after completion uses a fresh
attempt key. Transfer completion, server verification and worker preprocessing
are separate stages. Selected import detail refreshes while workers progress.

Geodata persistence no longer writes whole service snapshots. Follow the
[Phase 1 write-fenced migration/rollout procedure](../geodata/phase1-relational-authority.md#migration-and-rollout)
when updating API and worker images; obsolete writers are rejected instead of
overwriting current rows. The Phase 4 evidence permits only the charted
geodata API range of two to three replicas on the current cluster; it does not
authorize broader changes. Keep replica counts unchanged outside an explicitly
approved rollout until Phase 5 capacity/failure gates pass. See the
[Phase 4 evidence](../geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md).

For broker troubleshooting use the authenticated admin `/jetstream` page or
the [status/history API and runbook](messaging/jetstream-admin-status.md). Samples are
persisted by the operations service in `myota_core`; history does not depend
on a browser session or geodata pod cache.

Import history is durable and should be used as the operational source for
file visibility and processing status. The HTTP API only accepts durable work;
the outbox relay publishes preprocessing and promotion events to JetStream.
`geodata_import_worker.py` owns separate durable pull consumers with explicit
acknowledgement, one outstanding delivery per consumer per replica by default,
bounded redelivery, and PostgreSQL leases/heartbeats. The API has no local
durable-work executor. JetStream redelivers work when a worker exits before
acknowledging; atomic lease claims prevent simultaneous execution, and
idempotent candidate/entity identifiers prevent duplicate entities. The
`015_jetstream_worker_dispatch.sql` migration emits recovery events for queued
and processing records created before this worker became authoritative.
On SIGTERM/SIGINT each worker stops fetching new messages, finishes its active
delivery, and drains the NATS connection. Compose allows three minutes for
shutdown; Helm gives the worker a 180-second termination grace period. If the
worker is force-killed, JetStream redelivers the unacknowledged message and the
database lease/idempotent identity protects the retry.
Pasted KML/GPX uses its normalized GeoJSON recovery snapshot. Binary formats
without an installed parser remain visibly queued. Runs with no recoverable
source are changed to `FAILED` with `last_error`, so they do not appear
indefinitely as active work.

### Cancelling geodata preprocessing

The admin UI calls `PUT /v1/geodata/imports/{runId}/cancellation` for runs in
`UPLOAD_PENDING`, `QUEUED`, or `PROCESSING`. The endpoint is idempotent and
requires the existing `geodata.import` permission. Queued runs are cancelled
immediately; processing runs enter `CANCELLING` and the worker polls the
database, stops at a feature boundary, deletes staged candidates and source
objects, then records `CANCELLED`. The 202 response means that this checkpoint
is still pending. A worker restart finalizes an outstanding `CANCELLING` run
instead of resuming it. A stale JetStream delivery after cancellation is
acknowledged without starting work. Preprocessed runs cannot be cancelled;
administrators should use import finalization after review instead.

The import-processing worker also performs a lease-aware recovery sweep every
30 seconds (configurable with `GEODATA_CANCELLATION_RECONCILE_SECONDS`). It
finalizes only `CANCELLING` runs whose lease has expired or was never assigned,
so a lost worker or event cannot leave an upload in the active queue forever;
runs with a live lease remain owned by their current worker.

Cancellation timestamps, actor and completion status are written by the row
repository in the same transaction as the cancellation event. There is no
parallel SQL timestamp update. Finalization reloads the locked run before
discarding unfinished worker changes, then deletes staged records and persists
the terminal state while holding that lock. Repeating the request preserves
the original cancellation actor/time and returns the current status; a worker
heartbeat or stale projection does not require an administrator to reload.
Staged cleanup deletes by the indexed `import_run_id` and discards only this
run's pending projections; it never enumerates other imports or decodes their
geometries. This keeps cancellation responsive when other datasets are large.

### JetStream event retention

**Current-state runbook:** the instructions below describe the deployed legacy
topology and are not the ADR-0008 target. Phase 1 contract/provisioner work is
in progress, but no live broker configuration or producer/consumer path has
changed. Do not run the target provisioner against the deployed broker until
the [migration plan](messaging/nats-event-migration-plan.md) gates and a reviewed
cutover procedure are complete. Current versus selected state is shown in the
[topology diagrams](../architecture/diagrams/nats-event-migration.md).

`MYOTA_EVENTS` is a file-backed JetStream stream with **Interest** retention,
not an event-history log. It retains a message while at least one durable
consumer whose subject filter matches has not acknowledged it. Once every
matching durable acknowledges, JetStream removes the message. This supports
fan-out to the Activity notification consumer and the subject-specific
geodata workers without keeping already-completed messages in broker storage.
See the official NATS [retention-policy
semantics](https://docs.nats.io/learn/jetstream/retention-policies).

Each outbox relay creates or validates the five supported durable consumers
before it starts publishing. This ordering is essential: Interest retention
removes a message immediately if no consumer filter covers its subject. The
explicit geodata work subjects are allow-listed against those durable filters;
add a consumer before introducing a new work subject. Generic domain events
route under `myota.events.>` and are covered by Activity notifications. All
consumers acknowledge only after their database side effects/checkpoints have
committed; NAKed or unacknowledged work remains eligible for redelivery.

The existing 30-day stream `max_age` remains a safety limit for a stuck or
unconsumed backlog. It is not the normal cleanup mechanism and is not a
guarantee of indefinite preservation. Acknowledged events are not available
for later replay after all interested consumers have acknowledged them; use
service-owned PostgreSQL state and controlled domain recovery procedures
instead. The broker-status UI reports live message counts and consumer lag so
stalled acknowledgements remain visible.

### Activity notification consumer rollouts

The Activity notifications deployment uses the JetStream pull durable
`activity-notifications-pull-v1` on `MYOTA_EVENTS`, with the filter
`myota.events.>`, explicit acknowledgements, a 60-second acknowledgement
window, at most 64 unacknowledged deliveries, and at most 10 deliveries per
message. Replicas can fetch from the same durable during scaling and rolling
updates. On SIGTERM, the worker stops fetching, completes its current
notification and drains NATS; Helm allows 60 seconds for termination.

The earlier `activity-notifications` push durable cannot be converted in place
because its delivery mode is immutable. The pull durable uses the unchanged
database consumer identity `activity-notifications` and the notification's
`event:{eventId}` idempotency key. Already processed events are acknowledged
without creating another notice. Provision new durable consumers before
publishing events they need; under Interest retention, a newly added consumer
does not receive messages already acknowledged by all existing interested
consumers. After all old pods have exited and the pull consumer is healthy,
remove only the obsolete `activity-notifications` broker consumer; keep its
database checkpoints and processed-event records. Do not manually purge the
event stream as a recovery action.

`myota-activity-service/tests/test_notification_consumer.py` proves two
overlapping bindings, acknowledgement and restart against isolated JetStream
(`NATS_TEST_URL`). The geodata relational suite proves immediate cancellation,
idempotent retries, and finalization from a stale worker against isolated
PostGIS (`GEO_TEST_DATABASE_URL`, an `*_tests` database).

Preprocessing replay is idempotent per `(import_run_id, ordinal)`: it updates
the existing staged candidate while retaining its database identity and review
fields, rather than attempting a duplicate insert. Tagged load-test cleanup
deletes only the selected run's relational rows and writes associated row-level
control/audit/outbox changes; it never rewrites a service snapshot or flushes
unrelated candidate rows from another active import. If cleanup removed some rows before
failing, retry the same exact-tag cleanup request safely.

### Geodata import retention

The `geodata-import-retention` worker runs once daily in Compose and as a
Kubernetes CronJob. It permanently removes source objects and import history/
log records after 30 days. A `PROCESSED` run ages from `processed_at`; queued,
upload-pending, processing, cancelling/cancelled, preprocessed, legacy
completed, and failed runs
age from their latest start, completion, or heartbeat timestamp. Active work
continues to be retained while heartbeats advance; stalled imports and records
awaiting review expire after 30 days without activity. This includes failed
imports after 30 days, so preserve diagnostics externally if a longer failure
investigation period is required.

Cleanup removes source files, the import run/summary, snapshot manifests, and
published import-run outbox/consumer logs. It does not delete geodata entities
or their source provenance. The object is deleted before its database history;
if the object store or database cleanup fails, the run stays eligible and a
later run retries it. The chart settings `geodataImportRetention.enabled`,
`schedule`, `retentionDays`, and `batchSize` control Kubernetes. Local Compose
uses `GEODATA_IMPORT_RETENTION_DAYS` and drains all eligible runs in bounded
batches during each daily pass.

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
[`api-phase4-client-operational-migration.md`](../domain/api/phase-4-client-operational-migration.md).
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
[`observability.md`](../observability/overview.md) for the trust model and interpretation
of empty or unavailable series.

The **MyOTA Geodata capacity baseline** dashboard provisions request rate and
p50/p95/p99 latency, 5xx rate, active requests, request-body size, process CPU
and memory per service instance, Postgres pool/connection/lock-wait signals,
import queue age/heartbeat, feature/attempt totals, and unpublished geodata
outbox depth/age. Application metrics use bounded service/route/method/status
labels and do not label series with entity IDs, import IDs, or filenames.

The separate **MyOTA JetStream backlog and PostGIS query performance**
dashboard shows broker-reported consumer pending/ack-pending counts,
redeliveries, oldest outstanding message age and age-lookup availability, plus
PostGIS query-time percentiles and slow-query rate. These are distinct from
outbox rows waiting to publish and database import queue state. The
[`geodata load and query-evidence runbook`](../geodata/evidence/load-test-and-query-evidence.md)
documents profile safety, cleanup, metrics, and read-only query-plan capture.

The **MyOTA Object Storage** dashboard uses SeaweedFS's own S3 request
counters, server-side request-duration histogram, non-2xx status codes, and
in-flight upload count/bytes. Helm enables the private SeaweedFS metrics
listener on port 9324; it exposes both master and S3 metrics, and only the
Collector scrapes it. Do not configure a separate 9327 target: the deployed
SeaweedFS build does not listen on that port.
Do not publish either listener through an Ingress. A missing `up` series means
the scrape is unavailable, not that object storage had zero operations.

Grafana dashboard files are mounted from the observability ConfigMap using
`subPath`. Kubernetes does not refresh those mounted files when the ConfigMap
changes. The Helm chart therefore hashes all Grafana provisioning and dashboard
files into the Grafana pod template, causing Fleet/Helm updates to roll Grafana
and reload the files. If the SeaweedFS dashboard disappears or provisioning
logs report JSON `EOF`, compare the ConfigMap's `myota-object-storage.json`
entry with `/etc/grafana/dashboards/myota-object-storage.json` inside the pod;
a non-empty ConfigMap with a zero-byte mounted file indicates a stale pod that
needs the updated chart rollout.

### Permanent entity deletion recovery

Confirmed deletion jobs are durable rows in the geodata database, with their
event inserted into the transactional outbox in the same transaction. The
outbox publishes the geodata entity-delete subject to the shared MYOTA_EVENTS
JetStream stream. geodata-entity-deletion-v1 is the durable consumer name,
not a separate stream; zero broker-pending messages does not prove that every
database job reached a terminal state.

Deletion execution reloads each job from PostgreSQL before claiming it;
workers must not rely on a process-local snapshot for jobs created by the API
after worker startup. A missing job is an error and is never acknowledged as a
successful no-op. The geodata worker also periodically reconciles confirmed
QUEUED jobs and PROCESSING jobs whose lease expired, using the same database
claim as normal JetStream delivery. This repairs acknowledged/checkpointed
events that left a job unfinished and recovers work after worker restarts.
Inspect job statuses in geodata_control_record (kind=entityDeletionJobs)
alongside outbox and consumer checkpoints; do not manually purge the stream or
delete entity rows to recover a job. The Admin UI polls both individual and
bulk deletion jobs until they complete or fail, bounds each HTTP request, and
permits closing the dialog without cancelling a confirmed server-side deletion.
Bulk polling retries transient status-read errors without an overall timeout,
then closes the modal and reports terminal failures. The Entity Catalogue
page-size selector offers 25, 50, and 100 records.
For bulk requests the confirmation modal is rendered before per-entity impact
lookups begin; those lookups are batched and failures are shown per entity with
a retry action. Creating the preliminary jobs only gathers impact and creates
`AWAITING_CONFIRMATION` records. The outbox/JetStream deletion event is
published only after the administrator confirms in the modal, so no event is
expected while impact details are still being prepared.

After the 8 October 2026 rollout, the worker recovered the 45 queued jobs
observed during diagnosis; the database then had no queued or processing
deletion jobs. Two older `FAILED` legacy jobs remained with an instruction to
recreate and reconfirm them. Do not silently retry those legacy records; a
global administrator must create and explicitly confirm a new deletion job.

The repeatable external load tests use Grafana k6. The current
user-designated provisional-production target is the K3s deployment on
`spainip.es`, reached through `https://api.myota.top`. All performance/load
qualification profiles must target that deployment and run sequentially with
the documented hard caps, explicit production acknowledgement, dedicated test
account, and successful exact-tag cleanup. The read-only baseline creates no
application records; write profiles do, and their cleanup verifies there are
no linked activities or award progress. See the [geodata load-test
instructions](../geodata/evidence/load-test-and-query-evidence.md) and
[roadmap](../geodata/horizontal-scaling-roadmap.md#test-environment-premise--8-october-2026).

Unit/integration checks and destructive fault-injection, restart, termination,
queue-redrive, and untagged cleanup scenarios are not load profiles and must
remain in CI or an isolated non-production environment. The production load
policy does not authorize replica/configuration changes or service restarts.
The one-time Phase 4 API boundary test was separately authorized and is
complete; see the [Phase 4 evidence](../geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md).

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
Compose runs one geodata import worker by default; Helm configures worker
replicas independently from geodata API replicas. Each replica limits JetStream
ack-pending work to one message per consumer. The parser and some broad candidate
and spatial traversals still materialize large objects in worker memory, but
there is no authoritative whole-catalogue snapshot. Do not increase worker concurrency before completing the
remaining streaming/batched processing and provisional-production load evidence. A
large run should be allowed to finish before validation/finalization; do not
remove database or object-store volumes to recover from a transient outage.

Geodata entity lifecycle, geometry, and category edits are persisted in the
relational PostGIS tables. Migration 016 retains the old `service_state` row as
an archive; the durable runtime never reads or writes it. Request/job-scoped
repositories read authoritative rows and flush only changed rows. The service has no built-in entity
seed data; populate a local catalogue through the import or community-proposal
workflow instead.

## Production notes

Use managed PostgreSQL where possible, store credentials in Kubernetes Secrets
or an external secret manager, and back up `myota_core`, `myota_activity`, and
`myota_geo` independently. Only `myota_geo` needs PostGIS. Pin image digests,
enforce network policies so services reach only their own database, and expose
QGIS access only through a private network or bastion.

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

#### Fleet rollout and readiness troubleshooting

The gateway's liveness and readiness probes target `/healthz`; `/` is not a
health endpoint. The geodata import processor uses durable JetStream pull
consumers and may run multiple replicas; PostgreSQL leases and idempotency
protect side effects. Rollouts use `maxSurge: 0` and `maxUnavailable: 1`,
allowing a brief worker-capacity pause while unacknowledged messages remain in
JetStream and are redelivered after termination.

Fleet's GitRepo polling interval controls when a pushed revision is fetched.
A bundle force-sync/reconcile can re-apply the revision already fetched without
fetching a newer commit. When expected changes are missing, compare the GitRepo
observed commit with the pushed commit, check its polling interval, then inspect
the Bundle, Helm release, migration Job, pod events, and Deployment readiness.
The migration Job is release-revision-scoped and gates database-backed pods
until all three schemas are ready. A failed migration or unready pod should be
diagnosed from its logs/events; do not delete or reinitialize database PVCs as
a generic recovery step. See the detailed [deployment guide](https://github.com/myota-platform/myota-deploy/blob/main/deploy/helm/myota/DEPLOYMENT.md#rollouts-and-operational-checks).

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

The Helm observability stack is enabled by default and persists Prometheus,
Alertmanager, Grafana, and Tempo data on separate PVCs. Grafana is served only
at `/observability/` on the Admin UI host: the Admin UI first refreshes its
MyOTA access token, then its Nginx proxy validates that token against the
identity API for each Grafana request. Anonymous access and Grafana's own login
form are disabled; proxy-authenticated users receive the Grafana Viewer role.
Prometheus, Alertmanager, Tempo, and the collector have only cluster-internal
Services and no public ingress. See the
[deployment guide](https://github.com/myota-platform/myota-deploy/blob/main/deploy/helm/myota/DEPLOYMENT.md#rollouts-and-operational-checks)
for the path, storage sizing, and rollout checks. Alertmanager is deployed and
receives Prometheus/Grafana-managed alerts, but no email or paging destination
is configured by default; add an approved receiver before relying on external
notifications.

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
