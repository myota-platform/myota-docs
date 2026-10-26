# Operations

Use the [current system state](../architecture/current-state.md) for the
deployed revision, readiness, and operational evidence. The Operations Service
runtime and repository are retired; GitHub archived the repository on
2026-10-13 and [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3)
is closed.
See the [retirement record](evidence/operations-service-retirement-2026-10-11.md).

## Runbooks and infrastructure

- [Isolated test deployment](test-deployment.md) — `myota-test` Helm/Fleet
  release, initial live-data seed, endpoints, and testing-image update status.
- [Runbooks](runbooks.md) — local workflows, recovery, deployment, storage, and
  messaging procedures.
- [Database migrations and recovery](databases/README.md) — schema ownership,
  baseline history, and verified rebuild records.
- [PostgreSQL and object-storage backup/restore](database-object-backup-restore.md) —
  encrypted operator procedure, disposable-data rehearsal, and remaining
  RPO/retention decisions.
- [Fresh-database qualification](evidence/phase6-fresh-database-qualification-2026-10-26.md) —
  disposable-database migration and readiness evidence,
  including the new-volume follow-up.
- [Dependency-recovery qualification][phase6-dependency-recovery] —
  deployed single-instance
  NATS, PostgreSQL, and object-storage recovery evidence and limits.
- [Production core setup](production-core.md)
- [Storage operations](storage/README.md)
- [Messaging and NATS](messaging/README.md)
- [GitHub organization handoff](github-handoff.md)

[phase6-dependency-recovery]: evidence/phase6-dependency-recovery-qualification-2026-10-26.md

## Monitoring

- [Observability](../observability/README.md) — logs, traces, metrics, and
  dashboards.
- [NATS Surveyor](../observability/nats-surveyor-migration.md) — deployed
  monitoring topology and retirement decisions for former NATS samplers.
