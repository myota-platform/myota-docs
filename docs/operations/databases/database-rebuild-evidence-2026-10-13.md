# Core, Activity, and Geodata baseline rebuild evidence

**Verified:** 13 October 2026 11:08 UTC  
**Environment:** The non-production MyOTA namespace on the single-node K3s host  
**Status:** All three data-preserving rebuilds completed; no application schema or API behavior changed.

This record closes Deploy #16, Activity #5, and Geodata #9. Schema sources remain
owned by their service/repository owners; Deploy owns the combined migration
runner and Helm wiring; Platform remains a synchronized mirror.

## Baseline sources and deployed runner

| Database | Authoritative baseline on `main` | Git blob |
|---|---|---|
| `myota_core` | [Deploy `db/migrations/core/001_core_baseline.sql`](https://github.com/myota-platform/myota-deploy/blob/main/db/migrations/core/001_core_baseline.sql) | `deeec5f14c6d17d586421f58381d737c48b031d1` |
| `myota_activity` | [Activity `migrations/001_activity_baseline.sql`](https://github.com/myota-platform/myota-activity-service/blob/main/migrations/001_activity_baseline.sql) | `e87f84323e14cfa23af83e962edc7a2a1de54b83` |
| `myota_geo` | [Geodata `migrations/001_geodata_baseline.sql`](https://github.com/myota-platform/myota-geodata-service/blob/main/migrations/001_geodata_baseline.sql) | `b67b7d95fe2f28cb7c92ba8d233d0353d36e979f` |

The deployed immutable migration runner was
`ghcr.io/myota-platform/myota-service@sha256:f2e5a59ce23e275180aed8ad9482e209c42ff5c330a5ffda4b5e91088103cc51`.
Helm completed at revision 288. The runner records `baseline-v1` and sets
`myota_deployment_schema_state` to ready at revision 288 for each database.

## Final quiescent archives and restore rehearsals

Each archive was created after its database's writers and relays were drained.
The exact archive passed `pg_restore --list`, a full isolated restore, and a
data-only restore into the corresponding authoritative baseline. These
checksummed archives were stored outside Git with mode 0600 and removed after
acceptance evidence was recorded.

| Database | PostgreSQL / extension version | Bytes | SHA-256 |
|---|---|---:|---|
| Core | PostgreSQL 16.15 | 106,726 | `7f8526062da17d2ba8f045766d46049261ce62f884bce8dee6875b887c92079d` |
| Activity | PostgreSQL 16.15 | 864,682 | `e349be0b910e502a97744c935278b61f33a365667e4748ae5375ec696bc13baf` |
| Geodata | PostgreSQL 16.4; PostGIS 3.4.3 | 9,649,351 | `5a69e29ccc4395deeea38ead3e5a229350979ed313124a786110520b7835bff4` |

The full restores retained archive ACL entries. The baseline data restores omitted
`myota_schema_migration` and `myota_deployment_schema_state`; Geodata also
omitted `geodata_schema_feature` so the baseline's `row_authority_v1` feature
marker remained authoritative. Geodata's baseline writer-fence and geometry
triggers were disabled only while loading the trusted, previously validated
archive data, then re-enabled. A rollback-only application-role geometry update
confirmed the writer fence and derived Maidenhead geometry trigger worked after
restore. Runtime grants and default privileges were reapplied from the
Deploy-owned Postgres init policy.

## Quiescence and cutover sequence

For each database, the Fleet GitRepo was paused with drift correction disabled
while the owning API, worker, relay, and scheduled jobs were stopped. HPA/PDB
resources that could prevent an API scale-to-zero were temporarily removed for
the Geodata cutover. In-flight database sessions were drained; outbox rows were
published; work queues and durable deliveries were inspected before the archive.
After validation, Fleet was resumed and restored chart-managed resources.

| Database | Preserved data and catalog | Post-cutover validation |
|---|---|---|
| Core | 30 public tables, 72 entries in `public.pg_constraint`, 47 public indexes, `citext` 1.6, `pgcrypto` 1.3; 4 idempotency rows, 164 published outbox rows, and 2 service-state rows. The sole sequence remains unused. | Identity and Programme `/healthz` each returned 200 with durable mode. Service-state hashes matched the full-archive rehearsal. No unpublished outbox row or synthetic observability-correlation row remained. |
| Activity | 27 public tables, 57 entries in `public.pg_constraint`, 63 public indexes, `citext` 1.6; 155 notifications (all `DELIVERED`), 3,016 published outbox rows, 19,557 processed-event rows, and one consumer checkpoint. There are zero jobs, Activity work DLQ rows, generic DLQ rows, or unpublished outbox rows. | Activity `/healthz` returned 200 with durable mode. API 3/3, workers 2/2, notification consumer 1/1, relay 1/1, and retention schedule were restored. All six Activity work durables and the notification durable had zero pending or unacknowledged messages; the work stream had zero retained messages. |
| Geodata | 29 public tables, 75 entries in `public.pg_constraint`, 82 public indexes, four GiST indexes, and no invalid constraints or indexes. PostGIS, Tiger Geocoder, and Topology are 3.4.3; `pgcrypto` is 1.3 and `fuzzystrmatch` is 1.2. The restored data contains 10 entities/categories, 12 imports, 10 candidates, one completed import-processing queue row, 9,139 idempotency rows, 3,189 processed-event rows, five checkpoints, 15,929 published outbox rows, and zero source snapshots. The `row_authority_v1` marker remains present. | Geodata `/healthz` returned 200 with durable mode. Read-only entity listing and an authorized read of import status returned 200. The processing queue has no queued/running rows or active lease; the outbox has zero unpublished rows. All four Geodata durables and the `MYOTA_GEODATA_WORK` stream have zero pending/unacknowledged messages. Six unresolved database DLQ records were preserved. |

The earlier Core and Activity work-log comments reported 184 and 169
“constraints” without a reproducible counting definition. This record uses the
explicit catalog query `pg_constraint` restricted to the `public` schema;
the corresponding counts are 72 and 57. Both full and baseline restores report
all constraints valid. The older values are not used as acceptance counts.

## Runtime and deployment result

- Helm release `myota` is `deployed` at revision 288.
- Fleet GitRepo and BundleDeployment report Ready, 71/71 resources, and zero
  non-ready resources.
- Identity, Programme, Activity, and Geodata health endpoints returned HTTP 200
  with durable mode enabled. The Geodata entity-list and authorized import-read
  paths returned HTTP 200.
- Activity and Geodata worker streams have zero retained work messages. All six
  Activity work durables, all four Geodata work durables, and the Activity
  notification durable have zero pending and zero unacknowledged deliveries.
- Geodata's six unresolved application DLQ records remain intact for later
  owner-directed redrive. Published `geodata.loadtest.cleaned.v1` rows remain
  as cleanup audit history; no associated synthetic entity, import run, or
  queued work remains. The rollback-only trigger probe left no row change.
- The Geodata retention CronJob, two-to-three replica HPA, and PDB were restored
  by Helm/Fleet. All application Deployments are at their chart replica counts.

## Recovery procedure and cleanup

If validation had failed, the safe recovery action was to keep the affected
writers, consumers, and relay stopped; recreate the database from the exact
verified full archive; restore the documented runtime grants; and verify
readiness before resuming. No rollback was required. The checksummed archives
and isolated restore databases, client pods, temporary readiness Job, and
rehearsal namespace were removed after this evidence was captured. No database
test rows, temporary restore databases, test pods, or rebuild archives remain.

The three rebuild issue checklists are complete and closed:

- [Deploy #16 — Core rebuild](https://github.com/myota-platform/myota-deploy/issues/16)
- [Activity #5 — Activity rebuild](https://github.com/myota-platform/myota-activity-service/issues/5)
- [Geodata #9 — Geodata rebuild](https://github.com/myota-platform/myota-geodata-service/issues/9)
