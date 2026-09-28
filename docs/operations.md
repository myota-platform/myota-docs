# Local and production operations

## Local

1. Start Colima with Kubernetes enabled: `make colima-start`.
2. Run dependency-free tests: `make test`.
3. For the browser slice: `make run`.
4. For the durable local stack: `make compose-up`.
5. For Kubernetes: `make k8s-install`, then `kubectl -n myota get pods` and `kubectl -n myota port-forward svc/myota-gateway 8080:8080`.

The Compose stack is the supported local runtime for persisted data. It
supplies `CORE_DATABASE_URL` to identity, programme, activity and worker
processes, `GEO_DATABASE_URL` to geodata, and
`MYOTA_REQUIRE_DURABILITY=1` to every database-backed workload. A service
with a missing database URL exits during startup rather than using process
memory. The named PostGIS volume survives container restarts; do not use the
dependency-free `make run` process for data you need to keep.

`make test` remains dependency-free by design: unit tests explicitly exercise
the in-memory adapter and do not represent the production or Compose storage
path. Binary imports and award assets are stored in the mounted SeaweedFS volume;
their metadata, queues, audit events, QSO data and award state are persisted in
PostgreSQL.

### Import recovery

Import history is durable and should be used as the operational source for
file visibility and processing status. A queued or processing import has its
source document in SeaweedFS and an execution lease in the geodata
`import_run` table. Heartbeats keep active work leased; a restart or worker
failure leaves an expired lease that the next geodata instance requeues
automatically. A recovered run increments `attempt_count` and continues from
the original source. Runs with no recoverable source are changed to `FAILED`
with `last_error`, so they do not appear indefinitely as active work.

For diagnosis, inspect `status`, `attempt_count`, `heartbeat_at`,
`lease_until`, `last_error`, `filename` and `stats` in `import_run`, then check
the corresponding SeaweedFS object under the recorded bucket and object key.
Do not delete the PostGIS or SeaweedFS volumes while investigating an import.

## Production notes

Use managed PostgreSQL where possible, enable PostGIS, store credentials in Kubernetes Secrets or an external secret manager, and back up core and geodata databases independently. Pin image digests, enforce network policies so services reach only their own database, and expose QGIS access only through a private network or bastion.

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
