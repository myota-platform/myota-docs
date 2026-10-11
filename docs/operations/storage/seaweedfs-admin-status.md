# SeaweedFS monitoring in Grafana

**Current path:** OpenTelemetry Collector → Prometheus → provisioned Grafana
dashboard. The former Admin UI storage page, Operations sampler/API, and
sampled `operations_storage_snapshot` history are retired. This does not
change SeaweedFS object storage, application upload workflows, or application
health/readiness checks.

## Open the dashboard

Use **Admin → Metrics & dashboards** and open **SeaweedFS** at
`/observability/d/myota-object-storage`. Grafana is served through the
authenticated MyOTA gateway; the SeaweedFS metrics listener remains private
inside the cluster. The provisioned dashboard keeps the existing UID so saved
links continue to resolve.

The dashboard is a version-controlled adaptation of Grafana Labs dashboard
[19173, SeaweedFS](https://grafana.com/grafana/dashboards/19173-seaweedfs/).
Its Prometheus data source is `myota-prometheus`. The adapted panels use the
SeaweedFS series currently scraped by the Collector: scrape-target reachability,
S3 request rate/errors/latency, in-flight uploads, reported logical and
physical bucket size, volume-server filesystem gauges, and filer/volume request
rates. MyOTA-specific S3 panels remain where they add useful request and upload
detail. A panel is retained only when its query is backed by a deployed metric.

## Interpret the measurements

- `up` reports whether Prometheus can scrape the configured metrics endpoint;
  it is not an S3 transaction or end-to-end upload health check.
- Missing series are shown as unavailable or as gaps. They do not mean zero.
- Bucket gauges report buckets SeaweedFS exposes; they are not an object
  listing, and may omit empty or inactive buckets. Gauge freshness follows
  SeaweedFS's exporter update cadence.
- Volume-server filesystem gauges describe the backing filesystem. With
  K3s local-path storage this can be shared host capacity, not a PVC quota or
  MyOTA-only object usage.
- S3 counters can reset when SeaweedFS restarts. Grafana rates and histogram
  quantiles are estimates over the selected time window.
- The dashboard is an operational view, not an event archive or proof that a
  particular object is present. Use the owning API and object reference to
  investigate an upload.

## Ownership and configuration

`myota-deploy` owns the Collector scrape, Prometheus/Grafana provisioning,
dashboard JSON, alerts, Compose and Helm configuration. `myota-platform` carries
synchronized bootstrap mirrors. The private SeaweedFS metrics endpoint remains
scraped by the existing Collector; no separate Operations polling path or
database history is used. Operations retains bounded read-only PostgreSQL/PostGIS metrics and
current-identity Grafana role resolution. It has no NATS connection or
SeaweedFS monitoring API.

The dashboard source is
[`observability/grafana/dashboards/myota-object-storage.json`](https://github.com/myota-platform/myota-deploy/blob/main/observability/grafana/dashboards/myota-object-storage.json).
Deployment and provisioning guidance is in the
[operations runbook](../runbooks.md#seaweedfs-grafana-dashboard). Application
SeaweedFS credentials and readiness checks remain configured for the services
that store or retrieve objects; they are not dashboard credentials.

## Retirement record

The Operations storage sampler and API were removed, while the original
creation migration remains unchanged for migration-history integrity. A later
forward-only migration drops `operations_storage_snapshot` and its owned index;
the migration is idempotent. The dedicated Admin page is removed from
navigation and its former `/object-storage` bookmark redirects to Grafana.
SeaweedFS object-storage APIs, data, health checks, Collector scrape, and
Grafana alerts remain in service.

Implementation evidence and linked repository changes are recorded in
[myota-docs#5](https://github.com/myota-platform/myota-docs/issues/5) and the
cross-repository [SeaweedFS retirement issues](https://github.com/myota-platform/myota-deploy/issues/8).
