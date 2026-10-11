# NATS Surveyor monitoring consolidation

**Status (11 October 2026):** Phase 0 decisions are recorded. Surveyor 0.9.11
and the NATS dashboard set are present in the live cluster, with real metrics
confirmed. Source retirement has passed service CI and is committed. Deploy
is correcting and rolling out bounded JetStream capacity settings. The Admin,
Operations, and Geodata replacement images are not yet confirmed live; the
Operations history table remains until those images are healthy. This work is
not complete until the live exit checks below pass.

**Owners:** Volker Kerkhoff and Codex, the MyOTA platform team.
**Deployment authority:** `myota-deploy`; `myota-platform` is its synchronized
deployment and bootstrap mirror. Service changes belong to their owning repos:
`myota-admin-web`, `myota-operations-service`, and
`myota-geodata-service`.

## Decisions

- Run one internal NATS Surveyor replica. Scrape it every 15 seconds through
  Prometheus; keep the exporter Service cluster-local.
- Pin `natsio/nats-surveyor:0.9.11`. The ten Surveyor dashboard files are
  vendored from `nats-io/nats-surveyor` commit
  `0539d22d7bc7a4574c7f376f1dde37f00bfe2ee9`; Grafana dashboard 16256 is
  adapted to Surveyor metric names and the MyOTA Prometheus datasource.
- Keep all eleven dashboards together in Grafana folder `NATS`. Retain
  MyOTA-owned datasource and provisioning files. Provider deletion is enabled
  so removed source dashboards disappear from the persistent Grafana database.
- Use the existing system observer identity from Kubernetes Secret
  `myota-nats-surveyor-auth`. NATS clients and metrics stay inside the cluster;
  no public endpoint or Ingress is added. This does not change the application
  client trust boundary.
- Cap JetStream file storage at 7 GiB (7,516,192,768 bytes) beneath the
  8 GiB local-path PVC, and memory storage at 1 GiB (1,073,741,824 bytes).
  The local-path request does not enforce a quota. Live Surveyor initially
  reported a host-sized 120.56 GiB file-store capacity, 4 GiB reserved, and
  zero bytes used, so the host figure is not a safe alert denominator.
- Alert at 80% (warning) and 90% (critical) of configured file or memory store
  for ten minutes. Treat absent `up` or `nats_up` series as unavailable.
  Alert after two minutes. Do not set absolute queue thresholds from an empty
  broker.
- Retain Prometheus metrics for 15 days on its 10 GiB PVC. Surveyor is
  stateless and does not archive messages. JetStream stream retention remains
  the application’s replay/work policy, not a telemetry retention policy.
- Retire only duplicate NATS inspection. Keep Operations storage inspection
  and Grafana identity integration, Geodata business/worker metrics, and
  application outbox/worker delivery signals.
- Remove the old combined MyOTA broker/PostGIS dashboard and its two PostGIS
  panels. Do not recreate those panels elsewhere.

## Ownership and implementation evidence

| Concern | Authoritative owner | Deployment or mirror |
|---|---|---|
| Surveyor, NATS config, Prometheus scrape/rules, Grafana providers/dashboards, Compose and Helm | `myota-deploy` | Matching files in `myota-platform` |
| Admin JetStream route, view, client, styles and tests | `myota-admin-web` | Admin image |
| Broker sampler/API and snapshot schema ownership | `myota-operations-service` | `myota-deploy/services/operations.py` and `myota-platform/services/operations.py` |
| Geodata broker metrics startup and helper | `myota-geodata-service` | `myota-deploy/services/run_geodata.py` and `myota-platform/services/run_geodata.py` |
| Operations snapshot migration | `myota-operations-service/migrations` | Numbered core migration in Deploy and Platform |

Current source commits:
- Operations removal and CI repair: `601b9fc42013e98c8531320185b08f4fb756f940` (CI passed).
- Admin removal and stale browser assertion repair:
  `8c63f9ddc2f50091e975c836e5413e2bd6405c26` (CI passed).
- Geodata poller removal: `2a73101a741b8325afba9cda9176be0d85003e65` (CI passed).
- Deploy monitoring parity and bounded capacity:
  `221c44ccb0b1c5fe6ea894c7a2c4589be7685cd9` (Python and Helm checks passed).
- Deploy and Platform commit the same NATS capacity and restart fixes;
  Platform also mirrors the monitoring and runtime files.

The failed first NATS config rollout rendered byte limits in scientific
notation. NATS rejected the replacement before starting; the pre-existing
server stayed ready. Helm revision 203 failed while checking readiness due to
Kubernetes API client rate limiting. Revision 204 restored the prior release
without waiting, preserving the healthy broker. The corrected Deploy main
revision uses decimal string values and a `Recreate` strategy so one server
owns the persistent volume. Fleet reconciliation and the corrected rollout
remain required evidence.

## Dashboard compatibility

The Grafana folder contains dashboard 16256 and these ten upstream JSON files:
Clients, Clusters, JetStream meta-layer snapshot, JetStream state, NATS
Surveyor, NATS Overview, Network Usage, Node Resource Usage, Service
Observations, and Super Cluster.

A source audit compared 78 distinct NATS metric identifiers in the dashboard
queries with the live exporter. 48 were present; the remaining 30 belong to
route/gateway traffic, multi-server Raft/meta-layer state, and optional
service-latency observations that are not active in this single-server
deployment. Those panels must show no data; they must never convert missing
series to zero. Dashboard 16256's 13 core metric expressions are available.
The cluster, gateway, multi-node Raft, snapshot, and service-observation panels
are explicitly not applicable until those server features or observation
sources are deployed. Recheck Grafana query errors and representative panel
values against Prometheus after the source rollout. Do not claim every upstream
panel has live data on this topology.

The previous read-only synthetic check used an in-memory JetStream stream and
removed it afterward. The live exporter currently reports `nats_up=1`,
`nats_core_jetstream_filestore_used_bytes=0`, and
`nats_core_jetstream_memstore_used_bytes=0`. No test stream or pod is meant
to remain. Prometheus will retain metric samples for its configured 15 days;
this is expected telemetry history, not persistent test broker data.

## Phase status and exit criteria

### Phase 0 — decisions and access preflight

**Status: complete.**

- [x] Pin the Surveyor release and source dashboard revision.
- [x] Confirm the internal system observer secret and least-exposure model.
- [x] Set scrape interval, resources, retention, capacity limits and relative
  alert thresholds from the actual server/PVC limits.
- [x] Record the dashboard compatibility limits and rollback plan.

### Phase 1 — provision Surveyor and dashboards

**Status: deployed; panel verification has a remaining check.**

- [x] Deploy one Surveyor replica, internal metrics Service, Prometheus scrape,
  bounded JetStream collection, probes, resource requests and limits.
- [x] Provision dashboard 16256 and the ten pinned JSON files in Grafana folder
  `NATS`, retaining MyOTA-owned provider/datasource configuration.
- [x] Confirm real broker data is scraped and remove the temporary in-memory
  validation stream.
- [ ] Recheck all eleven provisioned dashboards in Grafana after the corrected
  source rollout, inspect query errors, confirm representative supported-panel
  values, and confirm unsupported single-node panels render no data rather
  than zero.

**Phase 1 exit:** Surveyor is connected and scrapeable, all eleven dashboards
load, supported panels return real metrics, unsupported topologies are called
out above, and missing data is not represented as zero.

### Phase 2 — retire duplicate monitoring

**Status: source changes committed; live rollout and schema retirement pending.**

- [x] Remove the Admin JetStream route/page/client/styles and retire its stale
  browser test expectation. Keep the general Observability workspace.
- [x] Remove the Operations broker sampler/API/metrics/dependency from the
  authoritative service. Preserve storage snapshots and Grafana identity.
- [x] Remove Geodata's duplicate NATS polling helper and startup hook.
- [x] Remove the old combined dashboard from source-managed Compose/Helm assets;
  use Grafana provider deletion for stale provisioned dashboards.
- [x] Replace Operations sampler alerts with Surveyor availability and bounded
  capacity alerts, including absent-series detection.
- [ ] Roll out the Admin, Operations and Geodata images through Fleet. Verify
  old routes, metrics and pollers are absent and retained Operations functions
  are healthy.
- [ ] Add and apply the Operations-owned forward migration to drop
  `operations_jetstream_snapshot` without `CASCADE`. The pre-migration
  custom-format backup contains 9,657 rows; it was verified with
  `pg_restore --list` and has SHA-256
  `634520fef35d62e647eb34b01d803efa7d77a91bf5e43eba7eda65118e215428`.
  Keep the backup outside source control. Do not drop the table until the new
  Operations image is live and storage history remains healthy.
- [ ] Confirm Fleet Ready, the corrected NATS config and capacities, all
  dashboard inventory, missing-series alerts, and no old snapshot table/index.
- [ ] Confirm Compose and Helm sources match Deploy/Platform mirrors.

**Phase 2 exit:** the old UI, Operations inspection/sampler and Geodata broker
poller are absent in live workloads; the snapshot table/index are removed by a
forward-only migration after the verified backup gate; NATS/Surveyor and all
supported dashboards are healthy; Fleet is Ready; and unrelated storage,
identity, outbox, worker, and domain behavior remains healthy.

## Phase 2 implementation prompt

```text
Complete Phase 2 of docs/observability/nats-surveyor-migration.md. Read this
roadmap, its evidence, the architecture diagram and AGENTS.md first. Work in
myota-admin-web, myota-operations-service, myota-geodata-service and
myota-deploy; keep myota-platform synchronized as the deployment mirror.

Deploy the CI-passing replacement images through Fleet. Verify the Admin
NATS route/page is absent, Operations no longer connects to or samples NATS,
the Geodata poller is absent, and Operations storage/Grafana identity functions
remain healthy. Confirm Surveyor, Prometheus, the eleven Grafana dashboards,
supported query results, alert health, configured capacities and Fleet
readiness.

Only after the Operations replacement is live, add its forward-only migration
to remove operations_jetstream_snapshot. The custom-format backup is already
verified; preserve it outside source control. Do not use CASCADE or remove any
other Operations, outbox, worker, checkpoint or domain table. Synchronize
Deploy and Platform migration copies. Update the Admin retirement notice,
architecture, repository links, work tracking and timeline with the exact
prompt and evidence. Test only through the host's local K3s environment,
remove ephemeral test resources, run focused CI/chart checks, and commit and
push each repository directly to main. Mark checkboxes only when each live
gate has passed.
```

## Remaining limitations

This migration removes duplicate monitoring; it does not make JetStream a
long-term event archive. Prometheus has 15 days of metrics retention. The
dashboard set includes upstream panels for cluster, gateway, Raft, and service
latency features that are not present in the current topology. The NATS event
and work migration remains a separate roadmap with its own stream retention,
replay, and database recovery gates.
