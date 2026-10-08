# Phase 3 prompt — bounded workers and failure recovery

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 3 — isolate preprocessing and promotion from API
pods** of the MyOTA Geodata API horizontal-scaling roadmap. Read the roadmap,
worker lifecycle diagram, Phase 1 authority record, and repository map first:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/diagrams/geodata-import-validation.md`
- `myota-docs/docs/geodata-phase1-relational-authority.md`
- `myota-docs/docs/repository-map.md`

Durable dispatch, separate workers, leases, retries, idempotent writes,
promotion, deletion, observability, and graceful NATS shutdown are implemented.
The remaining work is to make large-source parsing/candidate persistence
bounded, and to prove recovery during termination (including forced
termination) and concurrent multi-worker delivery.

Inspect current parsers, enrichment/conflation queries, candidate persistence,
checkpoints, lease handling, and tests. Implement end-to-end bounded processing:

1. Stream or incrementally decode supported large formats where feasible; do
   not retain the complete normalized feature set or full entity catalogue in
   worker memory.
2. Persist candidate/progress data in bounded batches with stable source
   identity and durable checkpoints. A restart or duplicate message must resume
   or replay safely without missing or duplicating candidates.
3. Use indexed, bounded database queries for location/proximity enrichment.
   Preserve existing PostGIS semantics and avoid loading all geometries into
   Python memory.
4. Keep entity, candidate-result, audit, and outbox writes atomic at each
   checkpoint where required by Phase 1. Discard uncommitted deltas on failure.
5. Add deterministic failure-injection coverage for termination during parsing,
   enrichment, candidate persistence, and promotion; include forced termination
   after the grace period and verify lease expiry/reclaim.
6. Exercise duplicate delivery and at least two workers claiming concurrently;
   verify one logical result, bounded retries/backoff, correct terminal error
   state, and no lost or duplicated work.
7. Verify SIGTERM/SIGINT stops new fetches, finishes or safely abandons the
   active unit, acknowledges only durable outcomes, and drains NATS.

Test representative large inputs and record a memory bound or measured peak
memory that demonstrates growth is bounded by batch/window size rather than
total input size. Keep the format-specific fallback explicit for any parser
that cannot stream; do not silently claim that format is bounded. Preserve
import ordering, validation, provenance, duplicate detection, review semantics,
and API response behavior. Avoid unrelated service-boundary changes.

Add reliable tests in the owning service and CI where practical. Use isolated
databases and brokers; never inject failures into production. Update worker
operations guidance and the roadmap with the test matrix and evidence. Check
items only when tests prove the stated failure behavior.

At the end, report processing memory behavior, supported streaming formats,
failure scenarios tested, multi-worker/replay results, changed files, validation
performed, and any format or recovery gate that remains open. Do not mark Phase
3 complete while parsing remains whole-run/in-memory or forced-termination and
concurrent-worker scenarios are unverified.

---
