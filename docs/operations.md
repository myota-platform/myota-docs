# Local and production operations

## Local

1. Start Colima with Kubernetes enabled: `make colima-start`.
2. Run dependency-free tests: `make test`.
3. For the browser slice: `make run`.
4. For PostGIS-backed containers: `make compose-up`.
5. For Kubernetes: `make k8s-install`, then `kubectl -n myota get pods` and `kubectl -n myota port-forward svc/myota-gateway 8080:8080`.

If Colima, kubectl or Helm is unavailable, `make test` and `make run` still work without external dependencies.

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
