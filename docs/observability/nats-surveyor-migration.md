# NATS and JetStream monitoring consolidation

**Status: proposed; no Surveyor deployment or legacy monitoring removal has
been implemented.** This page records the accepted direction and staged work
for replacing duplicate broker-inspection paths with NATS Surveyor and a
Grafana folder named `NATS` containing the requested dashboard set.

**Owner:** MyOTA platform work, maintained by Codex and Volker Kerkhoff.
**Deployment authority:** `myota-deploy`; `myota-platform` is its
synchronized deployment/bootstrap mirror. Service and UI changes belong to
their authoritative repositories.

## Goal and decision

Deploy one NATS Surveyor instance in the `myota` namespace. Prometheus scrapes
its internal metrics endpoint. Provision Grafana folder `NATS` with dashboard
16256 plus every dashboard JSON in Surveyor's
[`docker-compose/grafana/provisioning/dashboards`](https://github.com/nats-io/nats-surveyor/tree/main/docker-compose/grafana/provisioning/dashboards)
source folder: `clients-dashboard.json`, `clusters-dashboard.json`,
`jetstream-metalayer-snapshot-dashboard.json`,
`jetstream-state-metrics-dashboard.json`, `nats-surveyor-dashboard.json`,
`natsoverview-dashboard.json`, `network-dashboard.json`,
`noderesource-dashboard.json`, `observations-dashboard.json`, and
`supercluster-dashboard.json`. Pin and record the Surveyor release/commit used
for these vendored files; adapt its sample provider to the `NATS` folder and
retain MyOTA's existing Prometheus datasource UID and provisioning ownership.
Do not import Surveyor's sample datasource configuration over MyOTA's.

Also include [Grafana NATS Server dashboard 16256](https://grafana.com/grafana/dashboards/16256-nats-server-dashboard/).
Validate and adapt its PromQL before using it: Grafana describes it as designed
for the NATS built-in Prometheus exporter, while Surveyor exposes its own metric
families. Validate every dashboard panel against Surveyor metrics; an upstream
JSON file alone is not evidence of compatibility.

After Surveyor and the complete dashboard set pass the acceptance gates below,
remove the duplicate broker-inspection UI, APIs, samplers, Geodata broker
poller, broker-specific alerts, the combined MyOTA Grafana dashboard, and
database-backed NATS history.
Keep the Operations service for its SeaweedFS inspection and Grafana identity
functions. Keep application-level outbox and worker metrics: Surveyor reports
broker/server state and cannot tell whether a particular application relay
published its row or whether a worker committed its business effect. Retire
only the Operations broker-snapshot table and index; preserve transactional
outboxes, domain/work rows, idempotency/checkpoint state, dead-letter records,
and migration 021 recovery markers.

## Current state and authoritative evidence

As inspected on 11 October 2026, the following is current implementation, not
the target design:

| Current component | Authoritative source and synchronized deployment copy | Planned disposition |
|---|---|---|
| Admin UI NATS/JetStream page, route, API client, styles and browser tests | `myota-admin-web/src/router.ts`, `src/lib/adminNavigation.ts`, `src/views/JetStreamView.vue`, `src/lib/jetstream.ts`, `src/styles.css`, `tests/jetstream.test.ts`, and browser tests | Remove the page and menu entry after Grafana access and its role mapping are verified. Do not remove the general Observability workspace. |
| Read-only broker inspection, periodic sample API and snapshot persistence | Sources: `myota-operations-service/operations.py` and `myota-operations-service/jetstream_observability.py`; runtime copies: `myota-deploy/services/operations.py`, `myota-deploy/services/jetstream_observability.py`, `myota-platform/services/operations.py`, and `myota-platform/services/jetstream_observability.py`. | Remove only NATS-specific sampler, endpoints, metrics and routes. Keep Operations storage inspection, authentication/session integration and generic health/metrics. |
| NATS snapshot history | `myota-operations-service/migrations/001_operations.sql`; deployment mirror `myota-deploy/db/migrations/core/002_operations.sql`; bootstrap mirror `myota-platform/db/migrations/core/002_operations.sql`. | Retire the table and index through a forward migration after replacement metrics are verified and a pre-migration backup is confirmed. No broker payload or business-event archive is involved. |
| Per-Geodata-replica broker metric poller | Authoritative `myota-geodata-service/jetstream_observability.py`, `myota-geodata-service/run_geodata.py`, and `myota-geodata-service/tests/test_jetstream_metrics.py`; runtime copies `myota-deploy/services/jetstream_observability.py`, `myota-deploy/services/run_geodata.py`, `myota-platform/services/jetstream_observability.py`, and `myota-platform/services/run_geodata.py`. | Remove the broker polling helper and its startup hook after Surveyor covers the broker metrics. Preserve Geodata business, worker, outbox and import metrics. |
| MyOTA broker backlog dashboard | Compose: `myota-deploy/observability/grafana/dashboards/myota-jetstream-backlog.json` and `myota-platform/observability/grafana/dashboards/myota-jetstream-backlog.json`; Helm: `myota-deploy/deploy/helm/myota/observability/grafana/dashboards/myota-jetstream-backlog.json` and `myota-platform/deploy/helm/myota/observability/grafana/dashboards/myota-jetstream-backlog.json`. Helm provisions it from `myota-deploy/deploy/helm/myota/templates/observability.yaml`. | Retire after the requested `NATS` folder dashboards are validated. First move its two PostGIS panels to the Geodata capacity dashboard; do not lose query-performance visibility. |
| Broker availability and stale-sampler alerts | `myota-deploy/deploy/helm/myota/observability/rules.yml` and `myota-platform/deploy/helm/myota/observability/rules.yml` | Replace NATS-specific sampler alerts with Surveyor target/exporter health and useful broker/JetStream alerts. Keep unrelated service, worker and storage alerts. |
| NATS service and scrape configuration | `myota-deploy/deploy/helm/myota/templates/messaging.yaml`, `myota-deploy/deploy/helm/myota/values.yaml`, and `myota-deploy/compose.yaml`; synchronized copies in `myota-platform/deploy/helm/myota/` and `myota-platform/compose.yaml` | Add the Surveyor Deployment/Service and Prometheus scrape through Deploy, then synchronize the mirror. Current NATS container starts with JetStream and a data directory; the Helm template does not configure a system account or authentication. |

Do not remove the shared `observability.view` permission or Grafana role
mapping while retiring the NATS route; retain them wherever SeaweedFS or Grafana
still depends on them. Do not drop service outbox, work, checkpoint, dead-letter,
or Geodata migration 021 objects as part of removing broker telemetry.

The current Admin page and persisted snapshot are documented in the legacy
[JetStream status page](../operations/messaging/jetstream-admin-status.md).
The Phase 5 NATS work migration and durable retirement are complete within the
explicit timing waiver recorded in the [migration plan](../operations/messaging/nats-event-migration-plan.md);
that work does not remove these observability components.

## Target operating boundaries

- Run Surveyor as a single replica for the current single-server NATS
  deployment. Pin the image by version and immutable digest; do not deploy
  `latest`.
- Keep the NATS client and Surveyor metrics Services internal to the cluster.
  Do not create an Ingress, NodePort, or public scrape endpoint. Prometheus is
  the only intended scraper.
- Surveyor requires NATS system-account monitoring. Before rollout, qualify a
  narrowly scoped observer identity for the required `$SYS.REQ` requests and
  decide how that identity is provisioned. Current application clients connect
  without credentials. Do not silently turn on global authorization or place
  a system credential in ordinary Helm values; preserve current client
  connectivity and the accepted cluster-internal trust boundary.
- Begin with JetStream stats polling and a bounded metric set. Use leader-only
  collection and filter to required stream and consumer counters (messages,
  bytes, consumer count, pending, ACK-pending, redelivery and waiting pulls).
  Avoid detailed per-account metrics and unbounded labels unless a measured
  requirement justifies them. Do not subscribe to JetStream advisory subjects
  unless a specific alert requires advisories and a least-privilege permission
  has been reviewed.
- Vendor dashboard 16256 and all ten JSON dashboards from the Surveyor
  provisioning folder into the Deploy-owned Grafana dashboard assets, with
  upstream revision/commit, dashboard IDs, datasource UID and local query
  adaptations recorded. Provision all of them in Grafana folder `NATS`; keep
  datasource/provider configuration MyOTA-owned. Verify every panel against
  actual Surveyor metrics. If a dashboard cannot be made accurate with safe
  query changes, document incompatible panels and gate retirement of the
  existing combined dashboard until equivalent coverage is available.
- Configure alerts for exporter scrape/connectivity failure and actionable
  broker/JetStream conditions. Missing or stale series must remain distinguishable
  from zero backlog.
- Remove only duplicate broker sampling. Preserve `myota_outbox_nats_up` from
  `myota-deploy/services/outbox_worker.py`, outbox publish-failure/backlog
  signals, and worker outcome/retry metrics:
  these are application delivery signals, not substitutes for broker metrics.
- Retire the database history after the new metrics and requested dashboards
  pass direct comparison and the cutover acceptance checks. The current
  snapshots are operational samples, not an audit log. There is no
  stated requirement to preserve them as business history. Before the
  Operations-owned forward migration, take and verify a bounded database backup
  for rollback; then drop the table and index and synchronize deployment
  mirrors. Keep the backup under the normal database backup policy; no timed
  observation period is required.

## Phases and exit criteria

### Phase 0 — Compatibility and access preflight

**Status: planned. No runtime changes are authorized by this documentation
item.**

- [ ] Inspect the exact current Surveyor release/image, configuration flags,
  metric names and supported NATS server version; select a pinned image digest.
- [ ] Determine the minimum NATS system-account configuration and permissions
  Surveyor needs. Prove existing unauthenticated application clients can
  continue unchanged, or document the explicit migration needed before any
  production auth/config change.
- [ ] Pin dashboard 16256 and the Surveyor repository revision. Inventory all
  ten upstream dashboard JSON files and record their dashboard UIDs, required
  metrics, datasource assumptions and panel-to-metric compatibility. Specify
  the MyOTA-owned provider configuration for Grafana folder `NATS`; do not
  import upstream datasource provisioning.
- [ ] Identify where the two PostGIS panels from the existing combined
  dashboard will live after NATS dashboard retirement.
- [ ] Define exporter scrape interval, resource requests/limits, retention and
  alert thresholds from the current single-server capacity. Do not infer
  thresholds from an empty broker.
- [ ] Record the rollback procedure, direct broker/dashboard comparison checks,
  and backup verification needed before destructive cleanup. No timed overlap
  is required.

**Exit:** the chosen Surveyor build can connect with bounded permissions, the
Prometheus metrics are understood, dashboard compatibility is proven or
adaptation is scoped, and no existing service client or data boundary needs an
unreviewed change.

**Implementation prompt:**

```text
Work only on Phase 0 of docs/observability/nats-surveyor-migration.md.
Read that roadmap, the observability overview, repository map, Helm NATS and
Prometheus sources, and the current broker/dashboard evidence first. The
authoritative deployment repository is myota-deploy; myota-platform is its
synchronized mirror. Do not edit runtime configuration, deploy Surveyor, alter
NATS auth, or remove any current page, poller, alert, dashboard, endpoint, or
database object in this phase.

Qualify a pinned nats-surveyor release against the deployed NATS version and
document its exact bounded JetStream metrics. Inspect the source dashboard
16256 JSON and compare every query with Surveyor metric names. Determine the
least-privilege system-account access required without assuming that changing
the current unauthenticated client model is safe. Preserve application-level
outbox and worker metrics. Record exact sources, the compatibility mapping,
access decision, rollback steps, and any evidence gaps. Mark checkboxes only
when verified and commit documentation directly to main.
```

### Phase 1 — Deploy Surveyor and Grafana dashboards

**Status: planned; gated on Phase 0.**

- [ ] Deploy one immutable-image Surveyor replica and internal metrics Service
  through Deploy Helm; add it to the local Compose observability profile and
  sync authoritative templates/configuration to Platform.
- [ ] Provision the required observer access without exposing credentials in
  source, image, pod logs or ordinary values. Verify all existing NATS
  producers and consumers still connect.
- [ ] Add Prometheus scrape discovery and target health. Validate NATS core and
  JetStream stream/consumer metrics, including a nonzero disposable test
  workload in an isolated namespace; clean up test resources.
- [ ] Provision dashboard 16256 and all ten Surveyor-source dashboard JSON
  files in Grafana folder `NATS`, with MyOTA's existing Prometheus datasource.
  Confirm all ten dashboards and dashboard 16256 load. Validate relevant panels
  against real broker data; document any panel that is not applicable to this
  single-server deployment, and ensure no panel treats missing data as zero.
  Health alerts must fire for a controlled exporter failure in isolation.
**Exit:** Surveyor stays scrapeable and connected; all ten Surveyor dashboards
and dashboard 16256 load in folder `NATS`; every relevant panel agrees with
direct read-only broker metadata; missing data is not presented as zero; and
rollback to the existing observer is demonstrated. Phase 2 may begin as soon
as these checks pass; no timed overlap with legacy monitoring is required.

**Implementation prompt:**

```text
Implement only Phase 1 of docs/observability/nats-surveyor-migration.md after
its Phase 0 exit criteria are verified. Read the roadmap and recorded access,
metric and dashboard decisions. Implement in myota-deploy, synchronize
myota-platform, and change service-owned credentials only in the approved
secret/configuration path. Do not remove the legacy Admin page, Operations
sampler or history table, Geodata broker poller, alerts, or current Grafana
dashboard yet.

Deploy a single digest-pinned Surveyor internally, with bounded JSZ collection,
least-privilege monitoring access, a Prometheus scrape target, and dashboard
16256 plus all ten pinned Surveyor dashboards in Grafana folder `NATS`. Do not
add external NATS or metrics exposure. Test using
isolated synthetic broker data and remove that data afterward. Compare dashboard
values with read-only broker metadata, check scrape and dashboard health, and
record the cutover-validation results and rollback point. Preserve outbox and
worker outcome signals.
Run focused chart/render/configuration checks and deploy through Fleet only
after these checks pass. Report exact results, changed repositories, remaining
gaps and rollback evidence; mark only verified checkboxes.
```

### Phase 2 — Retire duplicate broker monitoring

**Status: planned; gated on Phase 1's dashboard, metric, permission, and
rollback acceptance checks. No seven-day overlap is required.**

- [ ] Remove the Admin UI `/jetstream` page, navigation, client code, styles
  and tests; remove NATS-specific Operations API endpoints, sampler, polling
  thread and NATS metrics. Keep the Operations service and SeaweedFS paths,
  Grafana identity integration, and shared permissions still needed by those
  features; remove only NATS-only permission usage.
- [ ] Remove the Geodata broker metadata polling helper and startup hook from
  its authoritative service source and synchronized runtime mirrors. Retain
  Geodata domain and worker metrics.
- [ ] Inventory provisioned and manually imported NATS dashboards by UID.
  Retire the combined `MyOTA JetStream backlog and PostGIS query performance`
  dashboard only after moving and verifying its two PostGIS panels in the
  Geodata capacity dashboard. Remove its Compose/Helm provisioning references
  and JSON copies from Deploy and Platform. Remove stale duplicate/manual NATS
  dashboards outside the managed folder. Provision dashboard 16256 and all ten
  pinned Surveyor-source dashboards together in Grafana folder `NATS`; retain
  the MyOTA-owned datasource/provider configuration and do not claim upstream
  dashboards are compatible until their panels are checked.
- [ ] Replace sampler-specific alerts with Surveyor health and broker alerts.
  Remove only alerts tied to the retired sampling endpoints/table.
- [ ] Add the Operations-owned forward migration to drop
  `operations_jetstream_snapshot` and its history index, then synchronize
  Deploy and Platform migration copies. Stop the sampler before applying it.
- [ ] Remove the Operations service's NATS-only dependency/configuration if no
  other Operations feature uses it. Keep generic service monitoring and
  SeaweedFS inspection.
- [ ] Update repository maps, READMEs, observability/messaging docs, diagrams,
  work tracking and public organization links. Retain this roadmap as the
  completion record.

**Exit:** no NATS broker-inspection UI/API/sampler or duplicate broker poller
remains; Grafana folder `NATS` contains dashboard 16256 and the approved,
source-managed Surveyor dashboard set, with no retired combined MyOTA NATS
backlog dashboard; the PostGIS dashboard remains and its two migrated panels
are verified; Operations still serves its non-NATS features; Prometheus
captures the replacement metrics; the old snapshot table/index are removed
through a verified migration; Deploy/Platform mirrors agree; and Fleet,
alerts, permissions and application NATS clients are healthy.

**Implementation prompt:**

```text
Implement only Phase 2 of docs/observability/nats-surveyor-migration.md.
Read its Phase 0/1 decisions and acceptance evidence first. The
owning repositories are myota-admin-web, myota-operations-service,
myota-geodata-service and myota-deploy; myota-platform deployment/runtime and
migration copies are synchronized mirrors. Do not change event/work processing,
NATS stream topology, application outbox publication, worker idempotency, or
unrelated Admin/Operations features.

Remove duplicate broker inspection only after Surveyor and the complete Grafana
dashboard set pass the recorded gates. Provision dashboard 16256 and every
JSON dashboard from the pinned Surveyor provisioning folder into Grafana folder
`NATS`, keeping datasource/provider configuration MyOTA-owned. Preserve
outbox/worker metrics and both SeaweedFS Operations inspection and
Geodata/PostGIS metrics. Move and verify the two PostGIS panels before retiring
the existing `MyOTA JetStream backlog and PostGIS query performance` dashboard. Retire the Operations
snapshot table with a forward-only owner migration and synchronized mirrors
after the sampler is disabled; do not hand-edit production schema or discard
history before the verified-backup gate. Verify no stale navigation, routes,
API clients, alerts or dashboard JSON remain, and verify the target dashboard
and non-NATS Operations paths after deployment. Update all docs and
repository links. Run focused tests and Helm/Fleet validation, clean isolated
test resources, record exact evidence, and mark only verified checkboxes.
Commit and push each repository directly to main with explicit messages.
```

## Acceptance reporting

The implementation record must list every changed repository/path, pinned
Surveyor image, NATS permissions and secret source, dashboard revisions/query
mapping, scrape/alert results, direct broker comparison and rollback evidence,
forward migration and verified backup, removed legacy components, preserved app
signals, test cleanup, and Fleet readiness. Do not mark this work complete from a successful Helm render
alone.
