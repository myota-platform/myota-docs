# Retired NATS / JetStream administration page

> The Admin UI `/jetstream` page, Operations broker sampler/API, and Geodata
> broker metrics poller have been removed from their authoritative source
> repositories. This page documents the retired behavior for historical
> reference. Follow the [NATS Surveyor migration status](../../observability/nats-surveyor-migration.md)
> for current broker evidence; the [SeaweedFS monitoring guide](../storage/seaweedfs-admin-status.md) documents the later retirement of Operations storage inspection and history.

The former page showed read-only JetStream broker metadata and persisted
periodic snapshots in `operations_jetstream_snapshot`. It did not consume
application work or change stream state. At the time of the NATS cutover, separate Operations storage inspection and
Grafana identity functions remained. SeaweedFS monitoring was subsequently
retired; only Grafana role resolution remains. See the current
[SeaweedFS monitoring guide](../storage/seaweedfs-admin-status.md).

The Operations-owned forward migration has removed the snapshot table and
history index after the replacement image and verified backup gates passed.
The backup remains outside source control. Do not use this historical page as
an operational runbook or infer broker health from its old sample description.
