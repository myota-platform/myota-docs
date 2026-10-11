# Observability documentation

- [Structured logging implementation roadmap](logging-implementation.md) —
  Phase 1–5 source/configuration plus a live Identity log/trace pair and
  successful dashboard query shapes. The PostgreSQL dashboard asset is mounted,
  while its Collector receiver is paused after a credential discrepancy.
  Authenticated Grafana UI navigation and Phases 6–7 remain open; Fleet
  reconciliation of Helm revision 231 is pending.
- [Logging implementation evidence (11 October 2026)](evidence/logging-implementation-2026-10-11.md) — current checks, image rollout status, and phase exit gates.
- [Logging operations runbook](logging-runbook.md) — LogQL queries, trace/log navigation, ingestion troubleshooting and capacity observations.
- [Logging phase prompts](prompts/README.md)
- [Metrics, traces, dashboards, and alerts](overview.md)
- [Geodata and PostGIS query evidence](../geodata/evidence/load-test-and-query-evidence.md)
- [NATS Surveyor and Grafana monitoring consolidation](nats-surveyor-migration.md)
  — live cluster monitoring set and dashboard compatibility limits; Phase 2 retirement is complete.
- [Retired JetStream status page](../operations/messaging/jetstream-admin-status.md)
  — historical behavior and current migration status.
- [SeaweedFS storage page and Grafana role mapping](../operations/storage/seaweedfs-admin-status.md)
- [Observability architecture diagram](../architecture/diagrams/observability-architecture.md)
- [9 October storage and Grafana verification evidence](evidence/2026-10-09.md)
