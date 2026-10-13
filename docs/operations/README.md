# Operations

Use the [current system state](../architecture/current-state.md) for the
deployed revision, readiness, and operational evidence. The Operations Service runtime and repository are retired; GitHub archived the
repository on 2026-10-13 and [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3) is closed.
See the [retirement record](evidence/operations-service-retirement-2026-10-11.md).

## Runbooks and infrastructure

- [Runbooks](runbooks.md) — local workflows, recovery, deployment, storage, and
  messaging procedures.
- [Database migrations and recovery](databases/README.md) — schema ownership,
  baseline history, and verified rebuild records.
- [Production core setup](production-core.md)
- [Storage operations](storage/README.md)
- [Messaging and NATS](messaging/README.md)
- [GitHub organization handoff](github-handoff.md)

## Monitoring

- [Observability](../observability/README.md) — logs, traces, metrics, and
  dashboards.
- [NATS Surveyor](../observability/nats-surveyor-migration.md) — deployed
  monitoring topology and retirement decisions for former NATS samplers.
