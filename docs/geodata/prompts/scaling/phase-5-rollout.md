# Phase 5 prompt — staged rollout and operational proof

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 5 — staged rollout and operational proof** of the
MyOTA Geodata API horizontal-scaling roadmap. This phase depends on the verified
recovery behavior in Phases 2 and 3 and the infrastructure qualification in
Phase 4. Start by checking those evidence gates and read:

- `myota-docs/docs/geodata/horizontal-scaling-roadmap.md`
- `myota-docs/docs/operations/runbooks.md`
- `myota-docs/docs/geodata/evidence/load-test-and-query-evidence.md`
- `myota-docs/docs/geodata/phase1-relational-authority.md`
- `myota-docs/docs/architecture/repository-map.md`

Do not declare the prerequisites complete merely because their roadmap boxes
are checked. Verify the linked evidence, image versions, and environment. If a
prerequisite is missing, record it and complete independent runbook, test, or
instrumentation work while leaving the dependent rollout gate open.

Build and execute a staged qualification plan using the current provisional
production deployment on `spainip.es` (`https://api.myota.top`) as the required
load/performance target. The operator has authorized bounded tests on this
cluster and cleanup of their exact tagged data. Run one bounded profile at a
time, preserve explicit production opt-ins, exact host allowlists, the
dedicated test account, and successful exact-tag cleanup. These results qualify
only this provisional deployment.

1. Add/pass CI failure-injection tests for forced worker recovery and API/object
   storage restart. Cover large-file upload, interrupted client transfer,
   abrupt receiver API termination, worker termination, duplicate event
   delivery, and SeaweedFS restart. Use the deployed storage image/version.
   Keep destructive process/storage failure injection isolated from production;
   a controlled rolling replacement of stateless API pods is separately
   permitted by step 3.
2. Run comparable bounded load tests against the provisional-production API.
   Retain sanitized evidence for throughput, p50/p95/p99 latency, errors,
   Postgres pool waits/connections, PostGIS plans/timings, object-store
   behavior, worker memory, and JetStream lag. Do not manually alter the HPA,
   its bounds, or replica counts to make a test pass; the currently deployed
   HPA may scale automatically within its Helm-configured limits.
3. During an accepted, run-tagged import, perform a controlled rolling
   replacement of only the stateless Geodata API Deployment, respecting its
   PDB and rollout strategy. Verify readiness, gateway health, durable upload
   recovery, terminal import status, and exact cleanup. Do not drain the only
   node or restart Postgres, SeaweedFS, or NATS in the live cluster; test those
   failure modes in an isolated environment.
4. Run a bounded provisional-production canary at the currently deployed
   topology. Observe errors, latency, database connection use, uploads,
   worker metrics, and queue lag through a representative window. Record
   thresholds, cleanup verification, and rollback triggers before starting.
5. Complete and drill the end-to-end rollback, in-flight import recovery,
   queue redrive, upload-session cleanup, and Phase 1 write-fence procedures.
   Verify the compatible code/schema rollback boundary and that no stale writer
   is reintroduced.
6. Update deployment limits, autoscaling bounds, dashboards/alerts, and
   operator documentation from measured results. Any manual production change
   to replica counts or HPA policy requires separate operator approval, a
   gradual plan, health checks, stop conditions, and a tested rollback.
7. Update the roadmap only with retained evidence and identify each environment
   and deployed version. Separate code/CI completion from runtime canary and
   production rollout evidence.

Production load/performance tests are authorized only for the designated
provisional target and within the bounded profile, account, and cleanup
controls above. This does not authorize destructive failure injection,
database/object-store/NATS restarts, node drain, queue redrive, untagged
deletion, manual HPA/replica changes, or unrelated rollout actions. Automatic
HPA scaling within its existing configured bounds and a controlled API roll
are allowed. Never expose credentials. If
account access, cleanup, or required metrics are unavailable, finish
reproducible automation and documentation; leave the runtime gate open and
provide the exact operator input needed.

At the end, report CI/failure coverage, replica-count results, canary evidence,
rollback drill result, changed files, dashboard/runbook updates, and any
remaining gate. Overall completion requires evidence that multiple API
replicas can roll, restart, and autoscale while imports recover, data remains
consistent, uploads are not pod-local, and throughput/latency improve without
unsafe downstream saturation. Do not claim production scaling completion
without the required staged evidence and authorized rollout.

---
