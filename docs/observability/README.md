# Observability

The [current system state](../architecture/current-state.md) is the entry point
for deployed monitoring components, verified dashboard state, and remaining
acceptance gates. Operations Service has been retired; its replacement
ownership and cutover evidence are documented in the
[retirement record](../operations/evidence/operations-service-retirement-2026-10-11.md).

## Guides

- [Overview](overview.md) — collection architecture, data sources, dashboards,
  alerts, and known gaps.
- [Logging implementation](logging-implementation.md) — roadmap, checklists,
  and acceptance criteria.
- [Logging runbook](logging-runbook.md) — provisioned Logs / Failures and
  Distributed Traces dashboards, LogQL, trace/log navigation, privacy limits,
  and ingestion troubleshooting.
- [NATS Surveyor](nats-surveyor-migration.md) — deployed exporter, Grafana
  dashboards, capacity decisions, and retired monitoring components.
- [SeaweedFS monitoring](../operations/storage/seaweedfs-admin-status.md) —
  Collector scrape, dashboard behavior, and metric limits.

## Evidence

- [Dashboard enhancements](evidence/dashboard-enhancements-2026-10-12.md)
- [Testing deployment verification](../operations/test-deployment.md)
- [Logging evidence, 12 October](evidence/logging-implementation-2026-10-12.md)
- [Logging evidence, 11 October](evidence/logging-implementation-2026-10-11.md) —
  historical rollout snapshot.
- [Identity and Programme metrics](evidence/identity-programme-metrics-2026-10-11.md)
- [Geodata TracerProvider repair](evidence/geodata-tracerprovider-2026-10-11.md)
- [NATS Surveyor cutover](evidence/2026-10-11-nats-surveyor-cutover.md)
- [Architecture diagram](../architecture/diagrams/observability-architecture.md)
- [Phase prompts](prompts/README.md)

The old JetStream status page is retained only for historical links. Use the
[Surveyor guide](nats-surveyor-migration.md) and
[current-state snapshot](../architecture/current-state.md) for current
monitoring ownership and runtime status.
