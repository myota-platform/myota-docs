# Phase 0 prompt — completed evidence review (archived reference)

> **Status: completed 8 October 2026.** The representative query/load evidence
> was captured at 2,875 entities and is documented in the [review artifact](../geodata-phase0-representative-query-review-2026-10-08.md)
> and [roadmap](../geodata-horizontal-scaling-roadmap.md#phase-0--establish-a-measurable-baseline).
> Keep this prompt as an audit reference; do not rerun production workloads or
> repeat permanent fixture provisioning unless a new, specific evidence gap is
> approved. This completion does not qualify larger entity/user/QSO scale.

The original task prompt is retained below for audit. The Phase 0 gate it
describes is complete; use the current roadmap/evidence, not this archived
instruction text, for current project status.

---

You are completing **Phase 0 — establish a measurable baseline** of the MyOTA
Geodata API horizontal-scaling roadmap. Work in the available MyOTA repositories
and use the current roadmap as the source of truth:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/geodata-load-test-and-query-evidence.md`
- `myota-docs/docs/geodata-load-test-upload-verification.md`

The read-only baseline, workload harness, telemetry, dashboards, and guarded
query-plan evidence tool already exist. The user has designated the current
K3s deployment on `spainip.es`, reached through `https://api.myota.top`, as
**provisional production and the sole target for load/performance
qualification**. The five bounded production write profiles and read-only
baseline were run on 8 October 2026; their sanitized results are in
`myota-docs/docs/geodata-phase0-production-evidence-2026-10-08.md`. Treat those
runs as delivered; do not repeat them without a specific evidence gap.
The former remaining gates—representative-cardinality PostGIS plans,
stable gateway telemetry labels, and measurable SeaweedFS operation timing—are
now reviewed in the linked representative query artifact.

First inspect the current harness, its safety checks, recent documentation,
repository instructions, the live provisional-production deployment, and its
load-test cleanup configuration. Reconcile profile names and limits from the
current implementation rather than relying on this prompt if they differ.
Run production load only against the exact approved host, one profile at a
time, with current hard caps, a dedicated test account, and successful
exact-tag cleanup. Never print or commit credentials. Do not run destructive
failure injection, restart/termination tests, queue redrive, or untagged
cleanup against production. If account access, cleanup, or stop conditions are
not safe, finish code/docs preparation and report the exact operator input
needed; do not simulate a passed run.

Complete the remaining gate by:

1. Review the retained production workload summaries and existing plans first.
   Do not rerun profiles unless the evidence review identifies a specific gap.
2. Capture additional bounded `EXPLAIN (ANALYZE, BUFFERS)` evidence only when
   representative, safely tagged cardinality is available, using the exact-host
   opt-in and short statement timeout. Review index use, actual versus
   estimated rows, buffers, and timings.
3. Correct and redeploy gateway environment/route telemetry labels, and add or
   expose SeaweedFS operation metrics. Correlate fresh production samples with
   the recorded workload windows; do not infer storage time from client upload
   duration.
4. Update the roadmap and evidence/runbook pages with links or durable,
   sanitized artifact locations, conclusions, and any remaining bottleneck.
   Check off only the gates directly supported by retained evidence.

Keep tests bounded by the existing profile caps. Make only necessary harness,
telemetry, dashboard, or documentation fixes demonstrated by evidence;
preserve the explicit production opt-ins, exact host allowlists, and cleanup
safeguards. Never scale or restart production services as part of a load run.
Before any code change, inspect the relevant owning repository and its local
instructions.

At the end, report the production profile summaries, query-plan evidence
locations, performance findings, changed files, validation performed, and any
gate that remains open with the reason. Historical non-production results are
not substitutes. This prompt was used to complete the Phase 0 review; its
results are documented in the linked artifact. Larger capacity goals remain
open under later roadmap phases.

---
