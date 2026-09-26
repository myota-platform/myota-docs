# ADR-0007: SeaweedFS as the default S3-compatible object store

## Status

Accepted

## Context

MinIO's former public community image is no longer a dependable distribution
path for the local and Kubernetes environments. MyOTA needs S3-compatible
storage for ADIF uploads, geodata source objects, award backgrounds,
signatures, and generated certificates. The application must remain portable
to another S3 provider and must not depend on an object-store-specific SDK.

## Decision

Use SeaweedFS as the default self-hosted object store. Services communicate
through the AWS S3 API using `boto3` with path-style addressing and configurable
internal and public endpoints. The local Compose stack runs the official
SeaweedFS image in single-node `weed mini` mode. The Helm chart can optionally
run the same single-node mode for development; production deployments should
use a separately operated SeaweedFS cluster or its supported Kubernetes
deployment pattern and provide the endpoint and credentials through a Secret.

The filesystem adapter remains available only for dependency-free unit tests.
`myota-deploy/scripts/migrate-local-object-store.py` preserves its
`bucket/object-key` layout when copying any existing development objects into
SeaweedFS. The migration is complete for the current checkout: no filesystem
adapter directory or stored object was present, and the former development
MinIO volume was empty.

## Consequences

- The service code is S3-provider-neutral and can use SeaweedFS, AWS S3, or
  another compatible provider without changing API contracts.
- Presigned URLs use `MYOTA_OBJECT_STORAGE_PUBLIC_ENDPOINT`, which allows
  containers to use an internal DNS name while browser uploads use a
  browser-reachable host.
- SeaweedFS has a different administration UI and on-disk format; existing
  MinIO object files cannot simply be attached to SeaweedFS and must be copied
  through S3 or the migration tool.
- The default Helm embedded mode is single-node and is not a production HA
  topology. Production values must point at a durable, backed-up deployment.
- Bucket names, object keys, checksums, and relational metadata remain stable,
  so certificates and import records do not need a domain migration.
