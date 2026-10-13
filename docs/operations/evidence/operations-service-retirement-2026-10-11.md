# Operations Service retirement — verification evidence

**Verified:** 11 October 2026 22:53 UTC. Deploy owns runtime configuration;
Platform is its synchronized mirror. No business data or NATS messages were
injected.

- Operations was removed from Compose, Helm/Fleet, Collector targets, API
  routing, image values, and image publishing.
- Contracts removed its observability API/client methods. Admin no longer
  calls Operations for Grafana auth. Gateway checks the live Identity session
  and role; Identity excludes the retired `operations.read` scope.
- Direct Collector receivers collect bounded PostgreSQL health metrics from
  Core, Activity, and Geo. Query samples, top queries, statement logs, and SQL
  text remain disabled; monitor roles are database-scoped and least-privilege.
- Receiver transport is cluster-internal plaintext because DB images do not
  enable TLS. Forward-only migrations removed Operations snapshot tables.

## Credential correction and live metrics

The three monitor Secrets had trailing newlines that PostgreSQL accepted but
Collector substitution removed. Runtime-secret files and Kubernetes Secrets
were normalized to 64 bytes without newlines; DB roles were synchronized. A
temporary client pod authenticated to all three databases and ran `SELECT 1`,
then was deleted. No password values or business data were recorded.

| Metric | Core | Activity | Geo |
|---|---:|---:|---:|
| `postgresql_backends` | 6 | 6 | 6 |
| `postgresql_commits_total` | 1,877,021 | 3,370,494 | 2,681,486 |
| `postgresql_db_size_bytes` | 10,107,927 | 14,072,855 | 108,794,339 |
| `postgresql_deadlocks_total` | 0 | 0 | 2 |

Collector logs contain no PostgreSQL authentication failures after restart.
These readings are samples, not capacity thresholds.

## Deployment and retirement checks

- Helm `myota-0.2.20`, revision 272, deployed; Fleet BundleDeployment Ready
  at `myota/myota:272`.
- 23/23 Deployments and 27/27 replicas are Ready; Operations Deployment and
  Service are absent.
- Gateway auth without a bearer token returns HTTP 401 and
  `Cache-Control: no-store`.
- Both Core Operations snapshot tables and role `myota_operations` are absent;
  Activity and Geo have no Operations-prefixed tables.
- Diagnostic pods were removed. Docker/Podman are absent, so Compose runtime
  execution was unavailable; source/config checks and live Helm were used.

Deploy `ced2df0d35c869b93c3b5fe1943f1c1eeed59e43`; Python quality/Helm
validation passed before a final UTC-only dashboard edit. Platform mirror is
`38bb3f5d7ce3e7ffe534b787261745e057ec0474`. Identity
`c563ecc0ee43b8623c4646da3bfcd021b53abc87` passed Ruff/build. Admin
`09f8e1db5b160cc2ef1458378f98217e3f4b6035` passed browser checks/build.
Operations `5952738366824a1a9cc494fd2e68737553d73f5e` passed quality checks
and removed image publishing. Contracts `fb564e3fed6ef4482c29a56b9d194332f5804ee7` removed routes and clients;
Contract Freeze run [#138](https://github.com/myota-platform/myota-contracts/actions/runs/38182971884)
passed on 11 October 2026 at 23:06 UTC.

## Repository archive verification

On 2026-10-13, the GitHub repository `myota-platform/myota-operations-service`
was archived with its default branch still `main`. A follow-up GitHub API read
returned `archived: true`. There were no open pull requests; issue #3 was the
repository's only open issue and was closed after the archive state was
verified. No runtime, database, or deployment changes were made as part of the
archive action.

[Operations issue #3](https://github.com/myota-platform/myota-operations-service/issues/3)
contains the final acceptance checklist and archive evidence.
