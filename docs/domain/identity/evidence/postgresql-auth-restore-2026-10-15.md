# Identity PostgreSQL restore evidence — 15 October 2026

## Scope

This non-destructive drill advances monorepo [Identity issue #36](https://github.com/myota-platform/myota/issues/36)
and [Deploy issue #35](https://github.com/myota-platform/myota/issues/35)
under [workstream #60](https://github.com/myota-platform/myota/issues/60).
It used only the isolated Helm-backed `myota-test` environment. No Compose or
Podman deployment was used, and no production system or database was accessed.

## Backup and isolated restore

The PostgreSQL 16 Core pod created a custom-format `pg_dump` of `myota_core`.
The archive was 82,291 bytes with SHA-256
`e8ebd57d39ad733663a0628cbdab9240ce8b6ed79ad715db8007d45232acbf48`.
It was written to a mode-0600 temporary file and was not added to source
control or retained after the comparison.

The archive was restored using `pg_restore --exit-on-error --no-owner
--no-privileges` into a uniquely named, temporary PostgreSQL database on the
same isolated Test instance. The source Core database was not modified. The
restore completed, and exact row counts matched for all selected authentication
and migration-state tables:

| Table | Rows in source and restored database |
|---|---:|
| `account` | 2 |
| `account_credential` | 2 |
| `account_role` | 2 |
| `auth_session` | 15 |
| `auth_revocation` | 1 |
| `account_recovery_token` | 0 |
| `security_event` | 7 |
| `identity_role_definition` | 8 |
| `identity_account_profile` | 0 |
| `identity_login_rate_limit` | 0 |
| `identity_oidc_provider_mapping` | 0 |
| `identity_auth_state_migration` | 1 |
| `service_state` | 2 |

The temporary restore database was dropped and its absence verified. The
temporary archive was deleted. Only table names, counts, archive size, and
archive hash were retained; no row values, email addresses, credentials, or
tokens were recorded.

## Helm migration and runtime state

At evidence capture, Helm release `myota-test` was `deployed` at revision 488.
Fleet's Testing GitRepo was Ready 1/1 at monorepo commit
[`c712615`](https://github.com/myota-platform/myota/commit/c7126156dc7b0897f2c555b410db349df7d4c529),
with 76/76 resources ready. The Core schema readiness row was `ready=true` at
schema release revision 486. The migration Job for revision 486 completed, and
the ledger contains `baseline-v1`,
`002_identity_auth_support.sql`, and
`003_identity_auth_backfill.sql` as applied. The Identity backfill marker
`service-state-jsonb-v1` records source counts of 2 accounts, 2 credentials,
2 roles, 13 sessions, and 7 events; target counts match, with the expected
additional migration audit event (8 events).

The chart runs the migration Job before application readiness: Identity's pod
has an init-container wait for the matching database `release_revision` and
`ready` marker. The migration Job uses the referenced Kubernetes database
Secret; the release and migration output contain no credential values. The
Identity Deployment was Ready 1/1 at the pinned image digest
`sha256:c7fd4ff2fc5af8a97535eab654bd5b65730240ce01d27602691749131b91fd72`.

The `002` and `003` migrations are additive and rerunnable. Helm rollback
restored the prior chart revision while leaving the successfully applied
forward schema intact; readiness continued to reference schema revision 486.
No down migration or destructive schema rollback was run. PR
[#95](https://github.com/myota-platform/myota/pull/95) corrected the baseline
checksum guard, and PR
[#96](https://github.com/myota-platform/myota/pull/96) refreshed Testing image
digests before the successful migration rollout.

## Limits

This proves a custom-format Core backup can be restored into a separate Test
database and that selected Identity table counts match. It is not an off-node
backup, a PVC-loss drill, a point-in-time recovery test, or a production
recovery guarantee. The archive and temporary restore database were deliberately
removed after verification.
