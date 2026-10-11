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

The Operations-owned forward migration has removed the snapshot table and
history index after the replacement image and verified backup gates passed.
The backup remains outside source control. Do not use this historical page as
an operational runbook or infer broker health from its old sample description.
