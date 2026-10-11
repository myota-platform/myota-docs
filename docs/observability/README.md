# Observability

The [current system state](../architecture/current-state.md) is the entry point
for the latest verified deployment and open acceptance gates.

- [Observability overview](overview.md) — collection architecture, data
  sources, dashboards, alerts, and known gaps.
- [Dashboard enhancement evidence](evidence/dashboard-enhancements-2026-10-12.md) —
  K3s, workflow, and PostgreSQL/PostGIS dashboards with live verification and
  metric limitations.
- [Logging implementation roadmap](logging-implementation.md) — phases,
  acceptance criteria, and remaining work.
- [Logging evidence](evidence/logging-implementation-2026-10-11.md) — dated
  rollout and verification record.
- [Identity and Programme metrics evidence](evidence/identity-programme-metrics-2026-10-11.md) —
  bounded instrumentation, scrape, dashboard, CI, and live K3s verification.
- [Geodata TracerProvider repair evidence](evidence/geodata-tracerprovider-2026-10-11.md) —
  duplicate initialization root cause, synchronized fixes, and host rollout.
- [Logging runbook](logging-runbook.md) — LogQL, trace/log navigation, and
  ingestion troubleshooting.
- [NATS Surveyor migration](nats-surveyor-migration.md) — exporter, dashboard,
  capacity, and Operations retirement decisions.
- [NATS Surveyor cutover evidence](evidence/2026-10-11-nats-surveyor-cutover.md).
- [Observability architecture diagram](../architecture/diagrams/observability-architecture.md).
- [SeaweedFS monitoring in Grafana](../operations/storage/seaweedfs-admin-status.md).
- [Phase prompts](prompts/README.md).

The old JetStream status page is retained only for historical links. Current
broker ownership is described by the Surveyor plan and the
[current-state snapshot](../architecture/current-state.md).
