# Phase 0 prompt — review remaining query and bottleneck evidence

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 0 — establish a measurable baseline** of the MyOTA
Geodata API horizontal-scaling roadmap. Work in the available MyOTA repositories
and use the current roadmap as the source of truth:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/geodata-load-test-and-query-evidence.md`
- `myota-docs/docs/geodata-load-test-upload-verification.md`

The read-only baseline, workload harness, telemetry, dashboards, and guarded
query-plan evidence tool already exist. **The write-profile execution gate is
complete:** every write profile has been run in a non-production deployment and
its result summary retained. The remaining roadmap gate is to review
representative query-plan/load evidence for PostGIS, object storage, and worker
bottlenecks.

First inspect the current harness, its safety checks, recent documentation,
repository instructions, and available non-production deployment configuration.
Reconcile the profile names and limits from the current implementation rather
than relying on this prompt if they differ. Do not target production, weaken
the harness guards, or use production runs as qualification evidence. Do not
print or commit credentials. If no safe non-production environment or required
credentials are available, finish the code/docs preparation and report the
exact operator input needed; do not simulate a passed run.

Complete the remaining gate by:

1. Reviewing the retained summaries for every currently defined write profile.
   Treat the completed non-production runs and retained summaries as delivered;
   do not rerun them unless the evidence review finds a specific gap.
2. Capturing bounded `EXPLAIN (ANALYZE, BUFFERS)` evidence on representative
   non-production data for the important map/catalogue queries. Review index
   use, actual versus estimated rows, buffers, and query timing. Correlate the
   workload results with PostGIS, object-storage, and worker bottlenecks.
3. Updating the roadmap and evidence/runbook pages with links or durable,
   sanitized artifact locations, conclusions, and any remaining bottleneck.
   Check off only the gates directly supported by retained evidence.

Keep the test bounded by the existing profile caps. Make only necessary
harness, telemetry, dashboard, or documentation fixes that are demonstrated by
the evidence; preserve safety caps and production opt-ins. Before any code
change, inspect the relevant owning repository and its local instructions.

At the end, report the reviewed profile summaries, query-plan evidence
locations, performance findings, changed files, validation performed, and any
gate that remains open with the reason. The write-profile execution checklist
item is already complete; do not reopen it. Phase 0 remains incomplete until
the representative query/load review has been documented.

---
