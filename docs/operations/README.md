# Operations

The Operations runtime is retired. Collector owns PostgreSQL telemetry, Gateway and Identity own Grafana authorization, and Surveyor owns NATS monitoring. Repository archive remains pending [issue #3](https://github.com/myota-platform/myota-operations-service/issues/3). See [retirement evidence](evidence/operations-service-retirement-2026-10-11.md).

- [Operations runbooks](runbooks.md) — local/production operation, JetStream,
  recovery, storage, and deployment procedures.
- [Database migrations and recovery](databases/README.md) — baseline ownership, runner verification, backup evidence, and rebuild gates.
- [Production core setup](production-core.md)
- [GitHub organization handoff](github-handoff.md)
- [Messaging and NATS](messaging/README.md)
- [Storage operations](storage/README.md)
- [Observability](../observability/README.md)
- [NATS monitoring consolidation plan](../observability/nats-surveyor-migration.md) — proposed Surveyor deployment and legacy monitor retirement.
