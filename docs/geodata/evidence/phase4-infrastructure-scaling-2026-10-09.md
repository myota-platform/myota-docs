# Phase 4 infrastructure and replica-safety evidence — 9 October 2026

## Result

**Phase 4 exit criteria are met for the documented deployment bounds:** one
geodata API pod was added and removed on the live Spainip K3s deployment without
violating the configured storage, JetStream, database-connection, or API
availability constraints. This evidence qualifies only a geodata API range of
two to three replicas on the current single-node cluster. It does not qualify
node failure, stateful-service failover, sustained throughput, or higher
replica counts; those remain Phase 5 work.

## Infrastructure review

- K3s had one Ready node (`spainip-k3s`), with approximately 12 allocatable CPU
  cores and 62.7 GiB allocatable memory. The cluster uses the
  `rancher.io/local-path` storage class.
- Geodata API and geodata processing worker pods mount no PVC and retain no
  accepted import source on pod-local storage. Import objects are in SeaweedFS;
  PostGIS and JetStream remain their own single-replica stateful services using
  `ReadWriteOnce` claims. Neither stateful service was scaled or restarted.
- PostgreSQL reports `max_connections=100`. Helm budgets the maximum API and
  processing-worker pools as `3 × 8 + 2 × 4`, plus 10 outbox connections and a
  40-connection reserve: **82/100 budgeted**, with 18 connections not assigned
  to those components. Helm rendering rejects worker replica or connection
  budget overrides that exceed the declared cap.
- API HPA: minimum 2, maximum 3, CPU target 70% of a 250m CPU request. The API
  has CPU/memory requests and limits, startup/readiness/liveness probes, a
  60-second termination grace, `maxSurge: 1` / `maxUnavailable: 0`,
  best-effort hostname spread and a PDB with `minAvailable: 1`.
- Processing worker is independently capped at two replicas, with a database
  pool of four per replica and explicit JetStream acknowledgements. It is
  scaled manually from the queue pending/oldest-message signals because this
  cluster has no validated external-metrics adapter. Phase 3 evidence covers
  competing consumers, duplicate delivery, database lease reclaim and graceful
  worker drain.
- Request concurrency remains bounded at 64 HTTP handlers per API pod and
  eight DB connections per pod. Upload parts are capped at 16 MiB and the
  request body at 1 GiB. There is no shared/global requests-per-second limiter;
  this qualification relies on the charted replica, handler and pool ceilings.

## Live boundary test

The task explicitly authorized this bounded production-cluster replica test.
No test user, entity, import, QSO, object, or broker event was created.

1. **Scale up:** Temporarily set the geodata HPA minimum to three. At
   `12:11:28Z`, desired was 3 and ready was 2; at `12:11:38Z`, desired and ready
   were both 3. The public gateway health endpoint stayed HTTP 200.
2. **Scale down:** Restored HPA minimum to two at `12:12:05Z`. By `12:12:15Z`,
   desired and ready were both 2, with the gateway still returning HTTP 200.
   The HPA returned to its configured `minReplicas: 2`, `maxReplicas: 3`; the
   PDB reported one available disruption.
3. **Database and broker:** After the test, the database reported seven active
   connections for the checked database, below `max_connections=100`. The
   `geodata-import-processing-v2`, `geodata-preprocessing-v1`,
   `geodata-entity-deletion-v1`, `activity-notifications-pull-v1`, and
   `geodata-location-enrichment-v1` consumers had zero pending messages, zero
   ack-pending, zero redeliveries, and zero oldest-message age. No processing
   worker or stateful workload replica count was changed.
4. **API pods:** The two ready API pods each used approximately 3m CPU and
   43–44 MiB memory at the final observation. This was a replica lifecycle
   check, not a load or CPU-HPA trigger test.

## Deployment and validation

Deployment change: [`myota-deploy` commit `a80257d`](https://github.com/myota-platform/myota-deploy/commit/a80257d36ac1d0046fd25b46bc6e0b1172902ef2),
which adds bounded API scaling, connection-budget validation, worker caps,
probes/resources/disruption handling, and matching local Compose pool limits.
The change was pushed to `main`; Fleet reconciled it and the cluster returned
to its configured replica counts. The GitHub checks passed:

- [Helm render and safety-budget workflow](https://github.com/myota-platform/myota-deploy/actions/runs/37928064009)
- [Deployment Python/repository quality](https://github.com/myota-platform/myota-deploy/actions/runs/37928065519)
- [Container build/publish workflow](https://github.com/myota-platform/myota-deploy/actions/runs/37928063987)

The Helm workflow rendered the chart and verified that configurations exceeding
the worker-replica or database-connection budgets are rejected. `git diff
--check` and the repository pre-push checks passed. Local Compose execution
could not be verified on this host because its Docker CLI does not provide the
Compose subcommand. Helm was rendered by GitHub Actions, not locally.

## Remaining limits

- The cluster has one K3s node. Best-effort topology spread cannot provide
  node-level availability, and local-path `ReadWriteOnce` data is not resilient
  to loss of that node.
- PostgreSQL, SeaweedFS and JetStream remain single-replica stateful services;
  their restart/failover and storage-loss behavior were not tested here.
- The boundary check did not apply sustained workload, exercise CPU-triggered
  HPA scale-up, test object-store saturation, or qualify greater-than-three API
  replicas. Phase 5 retains those gates.
- Worker auto-scaling is not enabled; it requires a validated external queue
  metrics adapter or a separately approved operational policy.
- No cleanup was required because the test changed only the HPA minimum and
  that setting was restored to two; no application or broker test data was
  created.
