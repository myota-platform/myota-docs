# Architecture decision records

This index links the decisions currently documented for MyOTA. ADR numbering
was inherited from parallel decisions and includes multiple `0007` files; do
not infer precedence from the numeric prefix. Repository/service ownership is
summarized in the [repository map](../repository-map.md).

## Platform and data boundaries

- [Storage topology](0001-storage-topology.md)
- [Three-database migration](0007-three-database-migration.md)
- [Geodata migration synchronization](0006-geodata-migration-synchronization.md)
- [Activity relational storage and workers](0005-activity-relational-storage-and-workers.md)
- [SeaweedFS object storage](0007-seaweedfs-object-storage.md)

## Services, policy, and administration

- [Service and repository shape](0003-service-and-repository-shape.md)
- [Programme policy ownership](0004-programme-policy-ownership.md)
- [Graphical geodata editing](0002-graphical-geodata-editing.md)
- [Administration roles and geodata controls](0007-administration-role-and-geodata-controls.md)

## Messaging and integration

- [NATS JetStream event and work topology](0008-nats-jetstream-event-and-work-topology.md)
