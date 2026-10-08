# Phase 4 prompt — infrastructure and scaling constraints

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 4 — remove unsafe infrastructure constraints** of
the MyOTA Geodata API horizontal-scaling roadmap. Read the roadmap, deployment
ownership map, operations guidance, and Phase 1 rollout fence before changing
deployment configuration:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/repository-map.md`
- `myota-docs/docs/operations.md`
- `myota-docs/docs/geodata-phase1-relational-authority.md`
- `myota-docs/docs/geodata-load-test-and-query-evidence.md`

The shared upload-spool volume and durable worker consumers are already in
place. Review and close the remaining gates: API pod volumes, concurrent
multi-worker qualification, database connection budgeting, query plans/indexes,
Kubernetes resources and availability settings, separate API/worker scaling,
and request backpressure under multiple API replicas.

Inspect `myota-deploy` Compose and Helm configuration plus the geodata service
pool/query settings. Use measured Phase 0/3 evidence where available. Then:

1. Inventory every API and worker volume, classifying whether it is required,
   durable, shared, or reconstructible. Remove or document unsafe API-local
   dependencies; do not remove durable object/database storage.
2. Qualify two or more workers under duplicate delivery, lease expiry, retry,
   and worker termination. Reuse Phase 3 tests/evidence; do not infer safety
   from consumer configuration alone.
3. Calculate the total Postgres connection budget across maximum API and worker
   replicas, pool sizes, migrations, operations, and other services. Set bounded
   per-pod pools and deployment maxima from that budget. Recommend PgBouncer
   only if evidence supports it; do not introduce it speculatively.
4. Review representative `EXPLAIN (ANALYZE, BUFFERS)` plans for high-volume
   catalogue/map queries. Add or adjust indexes only when measured plans and
   query workload justify them. Keep geodata migrations canonical in the
   service and synchronize platform/deploy mirrors byte-for-byte.
5. Set tested CPU/memory requests and limits, startup/readiness/liveness
   behavior as appropriate, graceful termination budgets, disruption budgets,
   and topology spread/anti-affinity for stateless API pods.
6. Configure API autoscaling from validated resource/latency signals and worker
   scaling from queue depth/oldest-message age. Define conservative min/max
   replicas from non-production evidence and preserve database/object-store
   capacity limits. Do not increase production replicas in this task.
7. Confirm rate limits, body/request limits, timeouts, queue backpressure, and
   failure responses behave correctly as API and worker replica counts change.

Keep Compose and Helm behavior aligned. Add or update deployment validation and
documentation. Run configuration rendering and repository checks, but do not
deploy or change production settings. If evidence for sizing is missing, encode
safe bounded values only when they are supported by explicit existing capacity
budgets; otherwise leave the scaling maximum conservative and document the
measurement needed to raise it. Do not guess resource numbers and present them
as measured.

Update the roadmap with evidence links and checkboxes supported by tests,
rendered configuration, and measured load data. At the end, report the volume
inventory, connection budget, autoscaling signals/bounds, manifests changed,
checks performed, and any constraint that remains open. Do not claim replicas
are qualified solely because the manifests accept a replica count.

---
