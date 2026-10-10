# NATS and JetStream monitoring consolidation

**Status: proposed; no Surveyor deployment or legacy monitoring removal has
been implemented.** This page records the accepted direction and staged work
for replacing duplicate broker-inspection paths with NATS Surveyor and a
Grafana NATS dashboard.

**Owner:** MyOTA platform work, maintained by Codex and Volker Kerkhoff.
**Deployment authority:** `myota-deploy`; `myota-platform` is its
synchronized deployment/bootstrap mirror. Service and UI changes belong to
their authoritative repositories.

## Goal and decision

Deploy one NATS Surveyor instance in the `myota` namespace. Prometheus scrapes
its internal metrics endpoint, and Grafana presents the NATS server and
JetStream metrics. Use the [NATS Surveyor project](https://github.com/nats-io/nats-surveyor) and
[Grafana NATS Server dashboard 16256](https://grafana.com/grafana/dashboards/16256-nats-server-dashboard/)
as the upstream components. Validate and adapt the dashboard's PromQL before
using it: Grafana describes that dashboard
as designed for the NATS built-in Prometheus exporter, while Surveyor exposes
its own metric families. The dashboard ID is not evidence of Surveyor
compatibility.

After Surveyor and the dashboard pass the acceptance gates below, remove the
duplicate broker-inspection UI, APIs, samplers, Geodata broker poller,
broker-specific alerts, old Grafana dashboard, and database-backed NATS history.
Keep the Operations service for its SeaweedFS inspection and Grafana identity
functions. Keep application-level outbox and worker metrics: Surveyor reports
broker/server state and cannot tell whether a particular application relay
published its row or whether a worker committed its business effect.

## Current state and authoritative evidence

As inspected on 11 October 2026, the following is current implementation, not
the target design:

| Current component | Authoritative source and synchronized deployment copy | Planned disposition |
|---|---|---|
| Admin UI NATS/JetStream page, route, API client, styles and browser tests | `myota-admin-web/src/router.ts`, `src/lib/adminNavigation.ts`, `src/views/JetStreamView.vue`, `src/lib/jetstream.ts`, `src/styles.css`, `tests/jetstream.test.ts`, and browser tests | Remove the page and menu entry after Grafana access and its role mapping are verified. Do not remove the general Observability workspace. |
| Read-only broker inspection, periodic sample API and snapshot persistence | `myota-operations-service/operations.py` and `jetstream_observability.py`; copies in `myota-deploy/services/` and `myota-platform/services/` | Remove only NATS-specific sampler, endpoints, metrics and routes. Keep Operations storage inspection, authentication/session integration and generic health/metrics. |
| NATS snapshot history | `myota-operations-service/migrations/001_operations.sql`; deployment mirror `myota-deploy/db/migrations/core/002_operations.sql`; bootstrap mirror `myota-platform/db/migrations/core/002_operations.sql` | Retire table and index through a forward migration after a verified overlap period. No broker payload or business-event archive is involved. |
| Per-Geodata-replica broker metric poller | Authoritative `myota-geodata-service/jetstream_observability.py` and `run_geodata.py`; runtime copies in Deploy and Platform. Tests: `myota-geodata-service/tests/test_jetstream_metrics.py`. | Remove the broker polling helper and its startup hook after Surveyor covers the broker metrics. Preserve Geodata business, worker, outbox and import metrics. |
| MyOTA broker backlog dashboard | `myota-deploy/deploy/helm/myota/observability/grafana/dashboards/myota-jetstream-backlog.json`; synchronized copy under `myota-platform/deploy/helm/myota/observability/grafana/dashboards/`. Helm provisions it from `myota-deploy/deploy/helm/myota/templates/observability.yaml`. | Remove after the Surveyor dashboard is validated. First move its two PostGIS panels to the Geodata capacity dashboard; do not lose query-performance visibility. |
| Broker availability and stale-sampler alerts | `myota-deploy/deploy/helm/myota/observability/rules.yml` and its synchronized Platform copy | Replace NATS-specific sampler alerts with Surveyor target/exporter health and useful broker/JetStream alerts. Keep unrelated service, worker and storage alerts. |
| NATS service and scrape configuration | `myota-deploy/deploy/helm/myota/templates/messaging.yaml`, `values.yaml`, and Compose configuration; Platform mirror | Add the Surveyor Deployment/Service and Prometheus scrape through Deploy, then synchronize the mirror. Current NATS container starts with JetStream and a data directory; the Helm template does not configure a system account or authentication. |

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
- Vendor the selected dashboard JSON with its upstream dashboard ID, revision,
  datasource UID and any local query adaptations documented. Verify every
  panel against actual Surveyor metrics. If the upstream dashboard cannot be
  made accurate with small query changes, create a source-managed MyOTA variant
  based on it and retain the upstream attribution.
- Configure alerts for exporter scrape/connectivity failure and actionable
  broker/JetStream conditions. Missing or stale series must remain distinguishable
  from zero backlog.
- Remove only duplicate broker sampling. Preserve `myota_outbox_nats_up`,
  outbox publish-failure/backlog signals, and worker outcome/retry metrics:
  these are application delivery signals, not substitutes for broker metrics.
- Retire the database history only after the new metrics and dashboard have
  been stable for at least the existing seven-day snapshot-retention period.
  The current snapshots are operational samples, not an audit log. There is no
  stated requirement to preserve them as business history; take a bounded
  pre-migration database backup for rollback, then drop the table and index
  through the Operations-owned forward migration and synchronized deployment
  mirrors. Keep the backup under the normal database backup policy.

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
- [ ] Fetch and pin dashboard 16256 JSON/revision. Compare its PromQL with
  Surveyor output and write a panel-to-metric mapping.
- [ ] Identify where the two PostGIS panels from the existing combined
  dashboard will live after NATS dashboard retirement.
- [ ] Define exporter scrape interval, resource requests/limits, retention and
  alert thresholds from the current single-server capacity. Do not infer
  thresholds from an empty broker.
- [ ] Record a rollback procedure and the seven-day overlap evidence needed
  before destructive cleanup.

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

### Phase 1 — Deploy Surveyor and Grafana dashboard

**Status: planned; gated on Phase 0.**

- [ ] Deploy one immutable-image Surveyor replica and internal metrics Service
  through Deploy Helm; sync the source to Platform.
- [ ] Provision the required observer access without exposing credentials in
  source, image, pod logs or ordinary values. Verify all existing NATS
  producers and consumers still connect.
- [ ] Add Prometheus scrape discovery and target health. Validate NATS core and
  JetStream stream/consumer metrics, including a nonzero disposable test
  workload in an isolated namespace; clean up test resources.
- [ ] Provision the selected dashboard JSON with the existing Prometheus
  datasource. Confirm dashboards render real broker data, health alerts fire
  for a controlled exporter failure in isolation, and no panel treats missing
  data as zero.
- [ ] Keep the existing Admin page, Operations sampler/table and legacy
  Grafana dashboard during a seven-day comparison period.

**Exit:** Surveyor stays scrapeable and connected; all required panels and
alerts agree with direct read-only broker metadata; the seven-day comparison
has no unexplained gaps or metric mismatches; and rollback to the existing
observer is demonstrated.

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
least-privilege monitoring access, a Prometheus scrape target and the selected
Grafana dashboard. Do not add external NATS or metrics exposure. Test using
isolated synthetic broker data and remove that data afterward. Compare dashboard
values with read-only broker metadata, check scrape and dashboard health, and
record the seven-day overlap start. Preserve outbox and worker outcome signals.
Run focused chart/render/configuration checks and deploy through Fleet only
after these checks pass. Report exact results, changed repositories, remaining
gaps and rollback evidence; mark only verified checkboxes.
```

### Phase 2 — Retire duplicate broker monitoring

**Status: planned; gated on Phase 1's complete seven-day overlap.**

- [ ] Remove the Admin UI `/jetstream` page, navigation, client code, styles
  and tests; remove NATS-specific Operations API endpoints, sampler, polling
  thread and NATS metrics. Keep the Operations service and SeaweedFS paths.
- [ ] Remove the Geodata broker metadata polling helper and startup hook from
  its authoritative service source and synchronized runtime mirrors. Retain
  Geodata domain and worker metrics.
- [ ] Move the PostGIS panels into the Geodata dashboard, then delete the
  previous MyOTA NATS backlog dashboard from Grafana provisioning and all
  authoritative/mirror JSON copies.
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
remains; Grafana has one source-managed NATS dashboard and the PostGIS
dashboard remains; Operations still serves its non-NATS features; Prometheus
captures the replacement metrics; the old snapshot table/index are removed
through a verified migration; Deploy/Platform mirrors agree; and Fleet,
alerts, permissions and application NATS clients are healthy.

**Implementation prompt:**

```text
Implement only Phase 2 of docs/observability/nats-surveyor-migration.md.
Read its Phase 0/1 decisions and seven-day comparison evidence first. The
owning repositories are myota-admin-web, myota-operations-service,
myota-geodata-service and myota-deploy; myota-platform deployment/runtime and
migration copies are synchronized mirrors. Do not change event/work processing,
NATS stream topology, application outbox publication, worker idempotency, or
unrelated Admin/Operations features.

Remove duplicate broker inspection only after Surveyor and Grafana pass the
recorded gates. Preserve outbox/worker metrics and both SeaweedFS Operations
inspection and Geodata/PostGIS metrics. Move the two PostGIS panels before
removing the existing combined Grafana dashboard. Retire the Operations
snapshot table with a forward-only owner migration and synchronized mirrors
after the sampler is disabled; do not hand-edit production schema or discard
history before the overlap and backup gate. Verify no stale navigation, routes,
API clients, alerts or dashboard JSON remain, and verify the target dashboard
and non-NATS Operations paths after deployment. Update all docs and
repository links. Run focused tests and Helm/Fleet validation, clean isolated
test resources, record exact evidence, and mark only verified checkboxes.
Commit and push each repository directly to main with explicit messages.
```

## Acceptance reporting

The implementation record must list every changed repository/path, pinned
Surveyor image, NATS permissions and secret source, dashboard revision/query
mapping, scrape/alert results, seven-day comparison, migration and backup
evidence, removed legacy components, preserved app signals, test cleanup, and
Fleet readiness. Do not mark this work complete from a successful Helm render
alone.
