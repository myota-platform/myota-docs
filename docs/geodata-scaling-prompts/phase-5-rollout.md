# Phase 5 prompt — staged rollout and operational proof

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 5 — staged rollout and operational proof** of the
MyOTA Geodata API horizontal-scaling roadmap. This phase depends on the verified
recovery behavior in Phases 2 and 3 and the infrastructure qualification in
Phase 4. Start by checking those evidence gates and read:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/operations.md`
- `myota-docs/docs/geodata-load-test-and-query-evidence.md`
- `myota-docs/docs/geodata-phase1-relational-authority.md`
- `myota-docs/docs/repository-map.md`

Do not declare the prerequisites complete merely because their roadmap boxes
are checked. Verify the linked evidence, image versions, and environment. If a
prerequisite is missing, record it and complete independent runbook, test, or
instrumentation work while leaving the dependent rollout gate open.

Build and execute a staged qualification plan using isolated non-production
environments first:

1. Add/pass CI failure-injection tests for forced worker recovery and API/object
   storage restart. Cover large-file upload, interrupted client transfer,
   receiver API termination, worker termination, duplicate event delivery, and
   SeaweedFS restart. Use the deployed storage image/version.
2. Run comparable bounded load tests at one, two, and increasing API replica
   counts. Retain sanitized evidence for throughput, p50/p95/p99 latency,
   errors, Postgres pool waits/connections, PostGIS plans/timings, object-store
   behavior, worker memory, and JetStream lag. Demonstrate whether scaling
   improves throughput without moving saturation downstream.
3. Deploy a two-replica non-production canary. Observe errors, latency, lost or
   duplicate work, database connection use, uploads, worker recovery, and queue
   lag through a representative operating window. Record thresholds and
   rollback triggers before starting.
4. Complete and drill the end-to-end rollback, in-flight import recovery,
   queue redrive, upload-session cleanup, and Phase 1 write-fence procedures.
   Verify the compatible code/schema rollback boundary and that no stale writer
   is reintroduced.
5. Update deployment limits, autoscaling bounds, dashboards/alerts, and
   operator documentation from measured results. Keep production replica
   changes as an explicit operator action with a gradual plan, health checks,
   stop conditions, and a tested rollback.
6. Update the roadmap only with retained evidence and identify each environment
   and deployed version. Separate code/CI completion from runtime canary and
   production rollout evidence.

Never run load, failure injection, cleanup, queue redrive, or rollout actions
against production unless the human operator explicitly authorizes that exact
action and target. This prompt itself does not authorize production changes.
Never expose credentials or delete untagged data. If live environment access or
approval is unavailable, finish reproducible automation, CI coverage, and
runbooks; leave the runtime gate open and provide the exact operator steps
needed.

At the end, report CI/failure coverage, replica-count results, canary evidence,
rollback drill result, changed files, dashboard/runbook updates, and any
remaining gate. Overall completion requires evidence that multiple API
replicas can roll, restart, and autoscale while imports recover, data remains
consistent, uploads are not pod-local, and throughput/latency improve without
unsafe downstream saturation. Do not claim production scaling completion
without the required staged evidence and authorized rollout.

---
