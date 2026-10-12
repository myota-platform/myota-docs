# Observability dashboard enhancements — verification evidence

**Verified:** 11 October 2026 22:07 UTC  
**Scope:** Deploy-owned Grafana and Helm changes, Platform mirror, and read-only
inspection of the local MyOTA namespace on K3s. No application runtime code
changed and no synthetic application or NATS data was injected.

## Delivered

The authoritative implementation is
[myota-deploy](https://github.com/myota-platform/myota-deploy), commit
[`226d9f6`](https://github.com/myota-platform/myota-deploy/commit/226d9f63cc08ce1852dbf2c04856feb45a9f0966).
The synchronized Helm source is
[myota-platform](https://github.com/myota-platform/myota-platform), commit
[`388e67e`](https://github.com/myota-platform/myota-platform/commit/388e67e6777c6e680f747fc4a38a0bf68ebf0819).

- **K3s workload health and capacity:** the Helm-only
  `MyOTA K3s workload health` dashboard displays deployment availability,
  restart changes, HPA current/desired/min/max, pod CPU and memory against
  requests, and persistent-volume use. Kubernetes-only views are not added to
  Compose.
- **Application workflows:** `MyOTA application workflows` combines the
  available three outbox relay series, Activity jobs and notifications,
  Geodata import queue/age, JetStream stream/server aggregates, and links to
  Surveyor detail dashboards. No payload or message identifier is exposed.
- **PostgreSQL/PostGIS:** the expanded dashboard shows active/max connections,
  database size, commits/rollbacks/deadlocks, Geodata query p95, pool waiters
  and wait duration, and lock waits. A text panel calls out transaction age,
  cache, and vacuum/freeze signals that are not collected.

The Compose and Helm workflow dashboards have the same Git blob
`55c7fc7de534975c4f4487bb47ff313fd0f46ab6` in both Deploy and Platform.
The K3s dashboard blob is
`94979f4ba82c9532997d98532887ea9b9d83e021`; PostgreSQL/PostGIS is
`2fc059374dc1f797e5f08cc64b67cb525cc76a43` in both Helm repositories.
The application workflow root Compose dashboard and Helm dashboard also share
the same blob. The K3s dashboard is intentionally Helm-only.

## Collection and safety

- kube-state-metrics v2.19.1 is restricted to the MyOTA namespace and the
  resource kinds needed by the panels. Its requests are 30m CPU/64Mi memory;
  limits are 100m CPU/128Mi.
- The Prometheus K3s scrape interval is 30 seconds. Kubelet resource and volume
  metrics use the internal HTTPS endpoint with the projected ServiceAccount
  token and CA validation. Metric relabeling keeps MyOTA namespace series.
  No metrics Ingress or public endpoint was introduced.
- MyOTA-scoped permissions are separated from node-scoped permission. Fleet
  rejects cluster-scoped RBAC in the MyOTA target namespace, so the minimal
  Kubelet `nodes` and `nodes/metrics` permissions are owned by the
  `spainip/infra` cluster bundle and bind only the MyOTA Prometheus
  ServiceAccount.
- Missing measurements are visibly described; the dashboards do not convert
  absent series into zero. Labels remain bounded and exclude SQL, identifiers,
  payload content, and user data.

## Verification

- GitHub main heads: Deploy `226d9f63cc08ce1852dbf2c04856feb45a9f0966`,
  Platform `388e67e6777c6e680f747fc4a38a0bf68ebf0819`, and infra
  `8f8bf8e83dfae5b4cd5db71f713054ec565f1ca8`.
- Deploy Python quality passed on the final commit; the dashboard-focused
  Ruff checks and tests passed with the repository-pinned Ruff 0.16.10. Helm
  lint/render validation passed on chart 0.2.20. Platform dashboard files and
  relevant assertions were synchronized and their file parity was checked.
- Helm release `myota`: chart `myota-0.2.20`, revision 265, status
  `deployed`.
- Fleet's `myota-deploy-deploy-helm-myota` BundleDeployment is deployed and
  monitored at the Deploy commit above. The K3s namespace had 24/24
  Deployments available and 28/28 replicas ready.
- kube-state-metrics and Prometheus were both 1/1 Ready. The provisioned
  Grafana ConfigMap contained the K3s dashboard, and Grafana logged successful
  dashboard provisioning.
- Read-only Prometheus queries returned `up=1` for kube-state-metrics and
  both Kubelet scrape jobs; workload readiness, restarts, pod CPU and memory,
  PVC availability, and HPA state returned live series. PostgreSQL, Geodata
  pool-wait/lock-wait, outbox, Activity/Geodata work, and NATS Surveyor
  metrics also returned live series for their supported sources.
- No test records or messages were created, so no cleanup was required.

## Known limits

The current NATS Surveyor metrics expose broker/server and stream-level state,
not the per-durable pending, ack-pending, redelivery, and oldest-message-age
series requested in the initial dashboard idea. The workflow dashboard names
this limitation and links to the NATS folder. Application database recovery
rows remain authoritative. The PostgreSQL collector does not expose
transaction-age, cache, or vacuum/freeze panels; the dashboard explicitly
lists these as unavailable. A single K3s node and single PostgreSQL primary
also mean no multi-node availability, replication, or failover panels are
claimed. This dashboard work does not qualify production capacity, change
alert thresholds, or close the separate structured-logging acceptance work.

## Closed enhancement issues

- [Deploy #11 — K3s workload health and capacity](https://github.com/myota-platform/myota-deploy/issues/11)
- [Deploy #12 — application workflow and JetStream delivery](https://github.com/myota-platform/myota-deploy/issues/12)
- [Deploy #13 — PostgreSQL and PostGIS database health](https://github.com/myota-platform/myota-deploy/issues/13)

## Current deployment recheck — 12 October 2026 01:26 UTC

This read-only recheck confirms the dashboard rollout remains healthy on the
current Deploy main revision. It supplements the original 11 October query
verification above; it does not claim fresh per-panel Prometheus query results.

- Helm release `myota` is chart `myota-0.2.20`, revision 275, status
  `deployed`.
- Fleet BundleDeployment `myota-deploy-deploy-helm-myota` is Ready,
  Deployed, and Monitored at Deploy commit
  [`c7b6956`](https://github.com/myota-platform/myota-deploy/commit/c7b6956ebd23a220eb359a9814d749d2b2d94b58).
  Fleet reports 71/71 resources ready, with zero missing, modified, or not-ready
  resources.
- The namespace has 23/23 Deployments and 27/27 desired replicas ready.
  Grafana is 1/1 Ready.
- The provisioned observability ConfigMap contains the K3s workload,
  application workflow, and PostgreSQL/PostGIS dashboard JSON. The separate
  NATS dashboards ConfigMap also remains present.
- No test workload, business data, or NATS message was created.

Deploy issues [#11](https://github.com/myota-platform/myota-deploy/issues/11),
[#12](https://github.com/myota-platform/myota-deploy/issues/12), and
[#13](https://github.com/myota-platform/myota-deploy/issues/13) were already
closed as completed. Their original implementation and live-query evidence
remains in the verification section above; this recheck confirms the active
Helm/Fleet deployment still includes the three dashboards.
