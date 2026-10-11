# Retired NATS / JetStream administration page

> The Admin UI `/jetstream` page, Operations broker sampler/API, and Geodata
> broker metrics poller have been removed from their authoritative source
> repositories. This page documents the retired behavior for historical
> reference. Follow the [NATS Surveyor migration status](../../observability/nats-surveyor-migration.md)
> for live image rollout and Operations history-table retirement evidence.

The former page showed read-only JetStream broker metadata and persisted
periodic snapshots in `operations_jetstream_snapshot`. It did not consume
application work or change stream state. The separate Operations storage
inspection and Grafana identity functions remain.

The snapshot table and its history index are still present until the
Operations-owned forward migration is applied after the replacement image is
healthy and the verified backup gate is met. Do not use this retired page as an
operational runbook or infer broker health from its historical samples.
