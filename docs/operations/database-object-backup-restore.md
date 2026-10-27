# PostgreSQL and object-storage backup and restore

This runbook describes the current MyOTA Next backup tools and their Phase 6
qualification limits. It does not configure a schedule or provide an off-node
backup destination. Do not treat a local test archive as a production backup.

Related work: [Phase 6 issue #10](https://github.com/myota-platform/myota-next/issues/10),
[backup/restore qualification #89](https://github.com/myota-platform/myota-next/issues/89),
and [fresh database setup #88](https://github.com/myota-platform/myota-next/issues/88).
The object-storage utility is introduced in
[myota-next PR #119](https://github.com/myota-platform/myota-next/pull/119).

## Scope and consistency boundary

The PostgreSQL tool backs up the three independent owner databases: Core,
Activity, and Geodata. It writes one PostgreSQL custom-format dump per owner
and a manifest with database names and SHA-256 checksums. Restore verifies the
manifest, dump checksums, archive readability, target database names, and
distinct owner targets before replacing database contents. Each owner restore
uses one transaction. The three databases are not restored atomically as a
group; a later owner failure does not undo an earlier successful restore.

The object-storage tool backs up only the bucket names explicitly supplied in
`MYOTA_BACKUP_BUCKETS`. It streams current objects into a local directory and
records keys, sizes, SHA-256 checksums, content headers, user metadata, and
object tags. It rejects versioned buckets because historical versions and
delete markers are not included. It does not copy bucket policies, ACLs,
lifecycle rules, or object-lock settings. Restore requires pre-created empty
destination buckets and an explicit source-to-destination mapping; it never
overwrites a non-empty bucket.

Neither tool creates a coordinated point-in-time snapshot across services.
Before a real recovery backup, put the deployment into a write-quiescent state:
pause API writes and background work that can change database or object-store
state, then keep that state until both backups finish. The object-store
enumeration is not atomic while objects are changing. Resume writers only
after verifying the archive. Record the backup completion time as the recovery
point. Until an operator-selected frequency is configured, there is no
guaranteed RPO.

## Credentials and encrypted artifacts

Load credentials only into the process environment from the secret store:

- `CORE_DATABASE_URL`, `ACTIVITY_DATABASE_URL`, and `GEO_DATABASE_URL` come
  from a database backup identity held in the secret store. On the current
  `myota-test` deployment, the runtime URL Secret cannot lock/read
  `myota_nest_schema_migration`; the migration URL Secret can dump the live
  databases, but its role is privileged. Production backup should use a
  dedicated least-privilege backup identity rather than the runtime
  application role or a broad migration/superuser role.
- `S3_ENDPOINT`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` come from the
  object-storage configuration and Secret.
- The `age` recipient used to encrypt an archive is public. Keep the matching
  private identity in the operator secret store, separate from the archive.

The tools do not put credentials into the dump or object manifest. The
PostgreSQL dumps and object files are plaintext until encrypted. Create the
working directory on a private encrypted filesystem, keep it mode `0700`,
encrypt the completed archive immediately with `age`, and remove the plaintext
working directory after checking the encrypted archive. Do not commit either
the archive or a private age identity.

Example structure; set credentials in the environment from the secret store
before running it:

```sh
umask 077
work_dir="$(mktemp -d)"
archive_path="${work_dir}.tar.gz.age"

npm run db:backup -- "${work_dir}/postgres"
MYOTA_BACKUP_BUCKETS="${MYOTA_BACKUP_BUCKETS:?set the explicit bucket allowlist}" \
  npm run object-store:backup -- "${work_dir}/objects"

tar -C "${work_dir}" -czf - postgres objects \
  | age -R "${AGE_RECIPIENT_FILE:?set the recipient file}" \
      -o "${archive_path}"
```

The PostgreSQL tool requires PostgreSQL 16 client tools. Set `PG_DUMP_PATH`
if `pg_dump` is not on `PATH`. Keep the tool output directory and generated
timestamped database backup directory with the encrypted archive until its
integrity check is complete.

## Restore procedure

1. Select an isolated restore target. For PostgreSQL, prepare three empty
   databases whose names match the backup manifest and whose service endpoints
   are distinct. For object storage, pre-create empty destination buckets.
2. Decrypt into a private temporary directory. Keep the private identity
   outside the archive:

   ```sh
   restore_dir="$(mktemp -d)"
   age -d -i "${AGE_IDENTITY_FILE:?set the private identity path}" \
     "${ARCHIVE_PATH:?set the encrypted archive path}" \
     | tar -xz -C "${restore_dir}"
   ```

3. Inspect the PostgreSQL manifest, then restore to the isolated database
   targets. The confirmation must exactly match the timestamped backup
   directory name:

   ```sh
   MYOTA_RESTORE_CONFIRM="RESTORE ${POSTGRES_BACKUP_DIRECTORY_NAME}" \
     npm run db:restore -- "${restore_dir}/postgres/${POSTGRES_BACKUP_DIRECTORY_NAME}"
   ```

4. Restore object data into the mapped, empty buckets. The confirmation must
   exactly match the object backup directory's basename:

   ```sh
   export MYOTA_RESTORE_BUCKET_MAP="${BUCKET_MAP_JSON:?set the source-to-target bucket map}"
   MYOTA_RESTORE_CONFIRM="RESTORE objects" \
     npm run object-store:restore -- "${restore_dir}/objects"
   ```

5. Compare the restored database rows and object checksums with the known
   backup state. Before switching an application to restored buckets, update
   its configured bucket names through the reviewed deployment process and
   verify object references with the owning service.
6. Remove temporary decrypted files after recording checksums and outcomes.

The PostgreSQL restore command is destructive to its configured target
databases, so never point it at a live deployment during a rehearsal. Object
restore refuses non-empty targets. If an object restore fails partway through,
discard and recreate only those disposable target buckets before repeating.

## Current test evidence

On 2026-10-26, the tools were rehearsed against disposable data on the isolated
`myota-test` services. No live application database or existing object bucket
was replaced.

- Created one disposable database on each Core, Activity, and Geodata
  PostgreSQL instance, inserted a marker row, and used the repository
  `db:backup` tool with the test deployment's owner-database endpoints. All
  three custom dumps had non-zero length and passed manifest checksum and
  archive-list validation. An incorrect restore confirmation was rejected.
- Mutated each marker, restored the three dumps twice with the repository
  `db:restore` tool, and confirmed the original values were recovered through
  the `myota_app` service credentials. One three-database restore took 1.32
  seconds at this small fixture size. The disposable databases were dropped.
- Created a separate disposable SeaweedFS source bucket and an empty restore
  bucket. Backed up one synthetic object with a Unicode key, changed the
  source object, restored the archive into the empty bucket, and verified its
  bytes, SHA-256, content type, cache control, and user metadata. A separate
  age-encrypted archive was decrypted for the restore rehearsal. Both buckets,
  the object, archive, and temporary private age key were removed.
- The current test workload deployments remained Ready, all five dependency
  StatefulSets remained Ready, and all test PVCs remained Bound.

This rehearsal demonstrates backup and restore mechanics on test services with
disposable data. It does not restore a full application database, verify
cross-service object references, measure realistic restore time, or demonstrate
recovery after PVC loss.

## 2026-10-26 live-owner snapshot and isolated restore check

A separate check used the existing `myota-test` owner databases and object
buckets as sources. It did not replace or modify existing application records.
The temporary plaintext snapshots were kept under the ignored local
`myota-next/runtime-secrets/` directory while the checks ran, then removed;
they were not committed.

- The object-store backup covered all six configured MyOTA buckets. The source
  inventory contained two objects (45,172,562 bytes total). The repository
  `tools/s3-backup.mjs` restore command restored them to six newly created,
  empty scratch buckets. Bucket counts, object sizes, and SHA-256 hashes
  matched the backup manifest. The scratch buckets were deleted after
  verification; the source buckets were left unchanged.
- PostgreSQL 16 `pg_dump` ran inside each existing owner database pod and
  produced custom-format dumps of `myota_core`, `myota_activity`, and
  `myota_geo`. A unique marker table with one row was added temporarily to each
  source, included in the dumps, and then removed from the sources. The dumps
  were restored with `pg_restore --exit-on-error --single-transaction` into
  same-named databases on a temporary PostgreSQL 16/PostGIS pod in a separate
  namespace. All three marker rows were present after restore, all three dump
  checksums matched their manifests, and exact counts for the pre-existing
  tables matched: 46 Core, 27 Activity, and 66 Geodata tables/partitions. The
  original databases had zero rows in those tables at the time of the check.
  The temporary namespace was deleted after verification.
- The restored databases and buckets were temporary qualification targets.
  Existing MyOTA workloads and PVCs remained in place. The source marker
  tables were confirmed absent after cleanup.

This check demonstrates native PostgreSQL dump/restore and the object-store
utility against the current test deployment. PostgreSQL was exercised with
direct `pg_dump`/`pg_restore`, not the Node `db:backup`/`db:restore` wrappers.
The database sources had no pre-existing table rows, and the two object files
are not a representative production volume. The database and object snapshots
were taken separately without quiescing all writers, so they do not establish
a coordinated recovery point or cross-service object-reference integrity.
This evidence does not qualify full-data recovery, realistic RTO, off-node
backup, or permanent PVC-loss recovery; keep issue #89 open for those checks.

## 2026-10-27 repository object-store utility check

The repository `tools/s3-backup.mjs` utility was also exercised against a
temporary synthetic source bucket and empty restore bucket on the test
SeaweedFS service. It backed up and restored one 51-byte object, then verified
the restored object content, SHA-256, length, metadata, and tag. The utility
rejected an incorrect restore confirmation. Both temporary buckets and the
local archive were removed after the check; no configured application bucket
was changed.

This is a tool-path check with one synthetic object. It does not add evidence
for a representative backup volume, a coordinated database/object recovery
point, cross-service reference integrity, off-node retention, or PVC-loss
recovery. Keep issue #89 open for those remaining qualification requirements.

## 2026-10-27 quiesced database and object restore

For a coordinated check, all 16 `myota-test` Deployments (21 replicas) were
scaled to zero while the repository PostgreSQL and object-store tools captured
the three owner databases and six configured buckets. The encrypted local
archive passed decryption and tar-integrity validation. Only the application
Deployments were paused; PostgreSQL, SeaweedFS, and JetStream remained up.

- Restored the Core, Activity, and Geodata dumps into three same-named
  databases on a separate disposable PostgreSQL 16/PostGIS instance. Exact
  counts matched for every table: 46 Core, 27 Activity, and 66 Geodata tables.
- Restored all six source buckets into six new, empty scratch buckets using an
  explicit source-to-target map. All five objects (45,173,156 bytes) were
  downloaded from the restored buckets and matched their recorded SHA-256 and
  size.
- The restored Activity database contains two ADIF import rows (one completed,
  one failed) and one QSO linked to an import. Both stored bucket/key
  references resolved to the corresponding restored ADIF objects through the
  explicit bucket mapping.
- Removed the temporary restore namespace, scratch buckets, encrypted archive,
  and local age identity after validation. Restored the original deployment to
  21 available replicas; Admin, Participant, and API health endpoints each
  returned HTTP 200.

The source snapshot was taken while application writers were stopped, and the
restore was isolated from application traffic. This verifies a quiesced,
test-sized database/object recovery point and the stored cross-store reference
mapping. The migration URL role used for the dump is privileged; provision a
dedicated least-privilege backup identity before relying on this procedure for
production. Off-node storage, a schedule and retention policy, production-sized
data, realistic RTO, and PVC-loss recovery remain unqualified.

## Open operations decisions

- Choose an off-node backup destination and verify it is isolated from the
  source storage failure domain.
- Define a schedule, retention window, encryption-key rotation and recovery
  procedure, and an accepted RPO/RTO. No automated schedule or retention policy
  is configured today.
- Rehearse a full, quiesced service backup and restore, including database
  references to objects, against an isolated target. Keep the source release
  and its PVCs untouched.
- Measure realistic database and object volumes. The synthetic restore timing
  above is not an operational SLO.

## Candidate launch policy (not approved or configured)

The following defaults make the remaining operations decisions reviewable. They
are a proposal for the single-instance `myota-test` topology; they do not
describe current deployment behavior and must not be used to claim an achieved
RPO or RTO.

- **Scope and trigger:** run one coordinated, write-quiesced backup of Core,
  Activity, Geodata, and the six configured object buckets every 24 hours. Run
  an additional on-demand backup before a database migration, bulk geodata
  operation, or release with a data migration. Keep writers paused until both
  database and object artifacts pass integrity checks.
- **Destination and failure domain:** copy completed artifacts to a dedicated
  versioned object-storage bucket outside the K3s host and its local storage
  failure domain. Deny public access and application-service access. Require a
  separate backup identity limited to writing and listing backup objects, and
  a separate restore identity for recovery. Do not use the MyOTA runtime or
  migration database credentials for scheduled backups.
- **Retention:** retain 14 daily recovery points and 8 weekly recovery points.
  Apply lifecycle expiration only after a newer artifact has passed its
  manifest, checksum, and decryptability checks. Preserve an additional
  pre-migration recovery point until the migration is accepted.
- **Encryption and keys:** encrypt each completed archive with `age` before
  upload. Keep the public recipient in deployment configuration and the
  private identity in the operator secret store, outside the cluster and
  backup destination. Review key access quarterly and rotate the recipient
  through an overlap period that keeps retained archives decryptable.
- **Recovery objectives:** use a 24-hour target RPO after the daily job is
  operational. Set no RTO target until restore time is measured with a
  representative data volume and the required application restart and
  validation steps. Alert if the most recent verified off-node artifact is
  older than 26 hours or if a scheduled run fails.
- **Restore rehearsal:** monthly, restore the latest complete recovery point to
  isolated databases and empty object buckets, verify cross-store references,
  and record elapsed time, data volume, and operator actions. Never rehearse by
  replacing the live test release.

Before adopting this policy, confirm that the selected object-storage provider
supports the required versioning, lifecycle, identity, and encryption controls
and provision its endpoint and credentials through the secret store. Until the
provider and credentials exist, no off-node copy, retention enforcement,
scheduled alert, or 24-hour RPO is active. Keep #89 open until those controls
are configured and a scheduled backup and isolated restore are verified.

Track these remaining checks in [issue #89](https://github.com/myota-platform/myota-next/issues/89).
