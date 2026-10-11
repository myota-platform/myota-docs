> **Historical note (11 October 2026):** This plan records the Surveyor cutover
> state before the separate SeaweedFS monitoring retirement. The earlier
> references below to Operations storage history and the old combined dashboard
> are historical. SeaweedFS monitoring now uses the Collector and provisioned
> Grafana dashboard; Operations storage sampling/API and its history table are
> retired. See the [SeaweedFS retirement guide](../operations/storage/seaweedfs-admin-status.md)
> and the linked [retirement issues](https://github.com/myota-platform/myota-deploy/issues/8).

# NATS Surveyor monitoring consolidation

**Status (11 October 2026):** Surveyor 0.9.11 and all eleven NATS dashboards
are live. The corrected broker limits are active, all three retired service
paths are on replacement images, the Operations JetStream history table and
index have been dropped by the forward migration, and storage history remains
healthy. A temporary in-memory validation stream was observed and deleted. On
11 October at 00:35:46 UTC, all thirteen dashboard 16256 expressions also
returned live Grafana data frames without query errors. Helm revision 246 is deployed and Fleet reports Ready for Deploy main commit
`f7d2e65edd5248d70376cf99ccc98c836b7f95e3`. Its bundle reports all 67 resources
ready and the applied deployment ID matches the desired ID. See the
[current-state snapshot](../architecture/current-state.md) for the live scope. The implementation
and Phase 2 exit criteria are complete.

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
| Operations snapshot migration | `myota-operations-service/migrations/003_retire_jetstream_snapshots.sql` | `db/migrations/core/005_retire_jetstream_snapshots.sql` in Deploy and byte-for-byte Platform mirror |

Current source commits:
- Operations removal and CI repair: `601b9fc42013e98c8531320185b08f4fb756f940` (CI passed).
  Current Operations source head: `3b4302e60105e00ebdc13fe14c5de96d8ed12af4`;
  the README update passed run `38098411907` and published the runtime digest below.
- Operations-owned snapshot retirement migration: `9dd93091b9abcb74fb5c047bbbd6a592edffd186` (CI passed).
- Admin removal and stale browser assertion repair:
  `8c63f9ddc2f50091e975c836e5413e2bd6405c26` (CI passed).
- Geodata poller removal: `2a73101a741b8325afba9cda9176be0d85003e65` (CI passed).
- Deploy monitoring parity and bounded capacity: `2a5b96a130e6a452118be45777ba93081f4701ec` (Python and Helm checks passed).
- Deploy migration mirror and platform-image digest: `bada0388a0e684f8fea87b15452f7df87eced170` and `29b5ea3700c0ada5d2d30a6c9162e49f8a07f277`.
  Deploy main `d0d2f50846fe2c9e239d46a48809860f4b98d7f5` records Operations
  image digest `sha256:b50b4666919e1f4046067129abd95993d371093088ea5e5f7633fd5ced163af4`.
- Platform mirror updates: `14b04a577ade08758283ca7a404643a0ed59624a`,
  `049c6522bb3093a2a74424e9d2777d6cb6659666`, and
  `15fc56f93586c1724cde65f3a8f674ad78aeaba2`; Platform main is
  `c81033e4fe694c1d6a3dab431024d18d39ee85d6`.

The first NATS config rollout rendered byte limits in scientific notation.
NATS rejected the replacement before starting, and the prior server stayed
ready. Helm revision 203 failed during readiness calculation because of
Kubernetes API client rate limiting; revision 204 restored the prior release.
The corrected chart was then applied from Deploy main with decimal string
limits and a `Recreate` strategy. Helm revision 205 brought up the 7 GiB file
and 1 GiB memory caps; revision 206 ran the retirement migration image. NATS
logs reported the configured limits, restored the three existing streams and
their 11 consumers, and reached Ready. Operations, Admin, and Geodata now run
the published replacement digests. Fleet initially retained stale applied-release metadata after the direct
recovery. Restarting the local Fleet agent and requesting one force sync let
Fleet apply the desired bundle at revision 208; its Ready condition and all
64/64 resource counts now agree with Deploy main.

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
sources are deployed. At 00:35:46 UTC on 11 October, all thirteen expressions
in dashboard 16256 were executed through Grafana's live /api/ds/query endpoint
against the provisioned Prometheus datasource. Each returned HTTP 200, one data
frame, and no query error; representative values are recorded in the
[evidence record](evidence/2026-10-11-nats-surveyor-cutover.md). This closes
the post-rollout query check. Do not claim every
upstream panel has live data on this topology or represent absent series as
zero.

The live validation created an in-memory stream `MYOTA_SURVEYOR_VALIDATION_20261011`
with a three-minute maximum age, published one synthetic 53-byte payload, and
confirmed Surveyor exposed the stream label and non-zero memory usage. The
stream and temporary `nats-box` pod were deleted; no validation stream or pod
remains. After restarting the stateless exporter, Prometheus now reports one
active NATS server ID and the configured 7,516,192,768-byte capacity; the
old server-ID sample has expired. `up` and `nats_up` are both 1. The exporter
reports 84 metric names, and the dashboard folder contains all eleven expected
NATS dashboards. A read-only Grafana search found no retired MyOTA JetStream
backlog or PostGIS query dashboard. Prometheus retains its normal 15-day
telemetry history, not broker messages.

## Phase status and exit criteria

### Phase 0 — decisions and access preflight

**Status: complete.**

- [x] Pin the Surveyor release and source dashboard revision.
- [x] Confirm the internal system observer secret and least-exposure model.
- [x] Set scrape interval, resources, retention, capacity limits and relative
  alert thresholds from the actual server/PVC limits.
- [x] Record the dashboard compatibility limits and rollback plan.

### Phase 1 — provision Surveyor and dashboards

**Status: complete.**

- [x] Deploy one Surveyor replica, internal metrics Service, Prometheus scrape,
  bounded JetStream collection, probes, resource requests and limits.
- [x] Provision dashboard 16256 and the ten pinned JSON files in Grafana folder
  `NATS`, retaining MyOTA-owned provider/datasource configuration.
- [x] Confirm real broker data is scraped and remove the temporary in-memory
  validation stream.
- [x] Recheck the live Grafana folder inventory (11 dashboards), Surveyor
  health/capacity series, and dashboard 16256 metric availability after the
  corrected rollout. All thirteen dashboard 16256 expressions returned a
  successful Grafana data frame with no query error at 00:35:46 UTC on 11
  October. The 48/78 metric-family audit remains topology-aware: unsupported
  gateway, multi-server, Raft and optional latency panels have no series and
  must remain no-data.

**Phase 1 exit:** Surveyor is connected and scrapeable, all eleven dashboards
load, supported panels return real metrics, unsupported topologies are called
out above, and missing data is not represented as zero.

### Phase 2 — retire duplicate monitoring

**Status: complete.**

- [x] Remove the Admin JetStream route/page/client/styles and retire its stale
  browser test expectation. Keep the general Observability workspace.
- [x] Remove the Operations broker sampler/API/metrics/dependency from the
  authoritative service. Preserve storage snapshots and Grafana identity.
- [x] Remove Geodata's duplicate NATS polling helper and startup hook.
- [x] Remove the old combined dashboard from source-managed Compose/Helm assets;
  use Grafana provider deletion for stale provisioned dashboards.
- [x] Replace Operations sampler alerts with Surveyor availability and bounded
  capacity alerts, including absent-series detection.
- [x] Roll out the Admin, Operations and Geodata replacement digests. Verify
  their Deployments are Ready; the Operations image no longer exposes NATS
  sampling and the Geodata image no longer starts its duplicate NATS poller.
  Operations storage history remains live and healthy.
- [x] Add and apply the Operations-owned forward migration to drop
  `operations_jetstream_snapshot` without `CASCADE`. Migration 003 is mirrored
  as core migration 005 in Deploy and Platform and passed Operations/Deploy CI.
  The pre-migration backup was verified with `pg_restore --list`, has SHA-256
  `634520fef35d62e647eb34b01d803efa7d77a91bf5e43eba7eda65118e215428`, and
  remains outside source control. Database verification shows the table and
  history index absent while `operations_storage_snapshot` retains 6,046 rows.
- [x] Reconcile Fleet and verify `Ready=True` for Deploy main commit
  `d0d2f50846fe2c9e239d46a48809860f4b98d7f5`, Helm revision 208, matching
  applied/desired deployment IDs, and 64/64 ready resources.
- [x] Confirm the Compose/Helm monitoring sources and migration/digest mirrors
  are synchronized between Deploy and Platform.

**Phase 2 exit: complete.** Runtime and database retirement gates passed,
including verified backup, retained storage history, healthy Surveyor, live
replacement images, the absent old history table/index, all eleven dashboards,
and Fleet Ready on the desired Deploy commit.

## Phase 2 implementation prompt (executed)

```text
Finish the remaining Phase 2 GitOps gate for the NATS Surveyor monitoring
consolidation. Read this roadmap, the evidence record, the architecture diagram
and AGENTS.md first. The replacement Admin, Operations, and Geodata images are
live; Operations migration 003 is mirrored as Deploy/Platform core migration
005 and has already removed only `operations_jetstream_snapshot` and its index.
The verified backup remains outside source control and Operations storage
history is healthy.

Inspect why Fleet `myota-deploy` reports `WaitApplied(1)` and retains stale
`status.appliedDeploymentID` despite Helm revision 206 matching the corrected
Deploy chart and all MyOTA workloads being Ready. Reconcile through Fleet so
its desired current `main` commit is reported applied and Ready. Preserve
release 206's configured limits, images, dashboards and migration result; do
not roll back to the broken scientific-notation config or modify unrelated
services. If a safe controller recovery is needed, use the local K3s host and
verify all resources afterward.

Recheck `up`, `nats_up`, the single active server capacity series, Grafana's
11-dashboard NATS folder, absence of the retired dashboards and temporary
validation data, the absent Operations history table/index, and retained
storage snapshot history. Update this roadmap, evidence, architecture, README,
todo, work-in-progress, backlog, changes and timeline with exact current
results and this prompt. Run focused read-only/chart checks, commit and push
docs to main, and mark the Phase 2 exit only when Fleet reports Ready for the
current Deploy commit.
```

## Remaining limitations

This migration removes duplicate monitoring; it does not make JetStream a
long-term event archive. Prometheus has 15 days of metrics retention. The
dashboard set includes upstream panels for cluster, gateway, Raft, and service
latency features that are not present in the current topology. The NATS event
and work migration remains a separate roadmap with its own stream retention,
replay, and database recovery gates.
