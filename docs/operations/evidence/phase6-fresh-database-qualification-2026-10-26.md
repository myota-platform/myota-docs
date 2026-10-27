# Phase 6 fresh-database qualification evidence — 2026-10-26

This record covers both disposable databases in the isolated `myota-test`
PostgreSQL instances and a separate migration rehearsal on newly provisioned
Core, Activity, and Geodata PVCs. The new-volume rehearsal did not start the
NestJS APIs or workers against those databases.

Related work: [myota-next issue #88](https://github.com/myota-platform/myota-next/issues/88),
[Phase 6 issue #10](https://github.com/myota-platform/myota-next/issues/10), and
[Observe qualification issue #90](https://github.com/myota-platform/myota-next/issues/90).

## Procedure and results

- Created disposable databases `myota_phase6_core_20261026a`,
  `myota_phase6_activity_20261026a`, and `myota_phase6_geo_20261026a` in the
  existing isolated Core, Activity, and Geodata PostgreSQL instances. No
  application database or PVC was replaced.
- Ran the repository Helm migration Job against those databases. It applied 21
  Core, 5 Activity, and 7 Geodata migrations. A second run reported all three
  schemas current.
- Held the Core migration advisory lock while starting a concurrent migration
  Job. The Job waited; after releasing the lock it completed with the schemas
  current.
- Started a temporary Helm release against the disposable databases using
  immutable image digests. Gateway, Identity, Programmes, Activity, and Geodata
  reached Ready after the permissions detail below was corrected.
- Verified that the deployed Identity and Programmes workloads reference the
  Core database URL, Activity references the Activity URL, and Geodata
  references the Geodata URL. The three service database boundaries remained
  separate.
- Removed the temporary release, the three databases, and the scratch URL
  Secrets after collecting the evidence. Existing PostgreSQL PVCs were
  preserved.

The PostgreSQL/object-storage backup checks are recorded separately under
[issue #89](https://github.com/myota-platform/myota-next/issues/89).

## New PVC migration rehearsal — 2026-10-26 23:58 UTC

- Created a separate `myota-issue10-fresh` namespace and installed the
  repository's `myota-infrastructure` chart with newly provisioned 10 GiB,
  20 GiB, and 50 GiB `local-path` PVCs for Core, Activity, and Geodata. The
  PostgreSQL image was pinned to
  `ghcr.io/myota-platform/myota-next-postgres@sha256:e1e22f4378280e67dc0c91493ef8fcdd421f1f37ae7c7f2af6bd860e093988b2`,
  and PostGIS to
  `ghcr.io/myota-platform/myota-next-postgis@sha256:fca1ef15923a2c45c84c3b758955bf73a024829a3fb92e99cfc6f8903338da0c`.
  No existing `myota-test` database, PVC, or application was changed.
- Installed the `myota-next` chart with its application components disabled
  and the migration hook enabled. The normal pre-install migration Job used
  tools image
  `ghcr.io/myota-platform/myota-next-tools@sha256:b0167e81b787ef0e0a7eea1cbd0954020a02faebc8b3549ee91f864c7de1ea65`,
  initialized the databases from empty volumes, and recorded 21 Core, 5
  Activity, and 7 Geodata migrations.
- Upgraded the migration-only Helm release to run the pre-upgrade hook again.
  It completed and the three migration ledgers remained at 21/5/7.
- Ran the deployed runtime-role-isolation utility in a temporary Job. It
  verified the Core, Activity, and Geodata runtime roles can access only their
  respective databases; cross-owner connections were denied. The check read no
  application rows.
- All three new PostgreSQL pods reached Ready. The temporary namespace and its
  PVCs were deleted after verification; no scratch resources remain.

The new-volume test verifies the database role-init scripts, Helm migration
hooks, schema counts, repeat migration, and owner isolation on newly provisioned
storage. The 2026-10-27 application-readiness follow-up below covers the APIs
and browser applications. NATS-backed workers remain unqualified on these new
databases, so #88 stays open.

## Limitation and follow-up

The disposable databases were created inside already initialized PostgreSQL
instances. The PostgreSQL init container's `020-create-app-role.sh` runs only
for the initial database on a new data volume, so its per-database default
privileges were not applied to these later-created scratch databases. Identity
initially failed with a permission error for `myota_app`. For this scratch test
only, grants and default privileges were added to the three disposable databases;
Identity then became Ready. The manual grants and all scratch databases were
removed afterward.

The earlier disposable-database setup is not equivalent to a new PostgreSQL
volume. The new PVC rehearsal above now covers initial role setup and migrations;
the later application check covers API and browser readiness. Keep the test
release isolated and preserve all existing `myota-test` PVCs.

## Fresh-database application readiness — 2026-10-27

- The image baseline was the live `myota-test` Helm release, chart
  `myota-next-1.1.6-testing.c10df2f22479.1`, revision 32, status `deployed`.
  The temporary app release used the immutable image values captured from that
  release. The relevant chart directories in the checkout matched
  `origin/testing` at qualification time. Its 14 application image digests
  were:

  - `activity`: `ec1628d3f27034359a21f273b480d8a4f724edc59dfc87f50f11d973a3f0b019`
  - `activity-notifications`: `62575d4b7132071755fcad311d9887f1157e807f935e4dc3fc3d22471bc3dd78`
  - `activity-worker`: `eb1e4bc419b2ee459e40734f99ce0465e9a3a824bb2751cc8564516e5e871411`
  - `admin-web`: `b92eb97b09bc91b47cbeb09340536e3aa97115d0f65428e0418759a985192a90`
  - `email-worker`: `ac010dda6dad59b65a9df932cf212c5200d352a3e880a21e93c7040f18fa8141`
  - `gateway`: `3efad20c67882d0aba3975f42372f5f50882dbb5433367cc423ae058c1194778`
  - `geodata`: `20a08a24ab0b30622a42358113e9e9e73247766b65f4f0e8db8c67692bcda252`
  - `geodata-importer`: `09fd8c74466ec7c56fb11d044901ceadbc7c260157c5104d562a66041f7045c4`
  - `geodata-worker`: `a4ce4f04aa79097e995eefddd5d4ee63013034c0ce60ad023a3c0d61aa08300f`
  - `identity`: `4e053a54daf9f36a3567d0e700c61131e01fa1d4ec4297a3d13af687743bade9`
  - `identity-maintenance`: `047c24e5af50a345dec336ec3f9960af643421056b55062a14da8ad192dba70f`
  - `outbox-worker`: `1782c25b138c3fb89912531e20f799bfd3a353e38fa24ad2df4643a91c03c814`
  - `participant-web`: `8777645271f6d0382d3281660918a233c023184735a6a5cc16be4b53f6a8e101`
  - `programmes`: `c2188951f407c224f9cc99164657bdb4aba0c6ea1771323705680422685d599d`

- Recreated the temporary namespace with new Core, Activity, and Geodata PVCs
  (10/20/50 GiB) and installed the repository infrastructure and application
  charts. PostgreSQL and PostGIS used the pinned images recorded above in the
  2026-10-26 procedure; the tools image was
  `ghcr.io/myota-platform/myota-next-tools@sha256:b0167e81b787ef0e0a7eea1cbd0954020a02faebc8b3549ee91f864c7de1ea65`.
  Migration hooks initialized the databases from empty storage; direct ledger
  queries returned 21 Core, 5 Activity, and 7 Geodata migrations.
- Installed a separate NATS 2.11.8 / Helm chart 1.3.12 instance with an 8 GiB
  JetStream PVC and 6 GiB store cap. The provisioner validated all 4 streams
  and all 11 configured durable consumers. The initial 512 MiB scratch-store
  cap was too small for the reviewed stream limits; after correcting it to
  6 GiB, the provisioner completed successfully. No live NATS stream was used.
- Started all launch-profile APIs, browser applications, and enabled workers:
  15 Deployments and all 20 desired replicas were available. This included
  Activity and Geodata workers, Activity notifications, all 3 outbox workers,
  and the email worker configured with the test SMTP sink and its CA Secret.
  PostgreSQL (3 pods) and NATS (1 pod, 2 ready containers) were healthy. No
  running pod had a restart or readiness failure in the final check. The
  Geodata source importer remained disabled as in the test profile. SMTP
  delivery itself remains covered separately by #13.
- The Gateway returned HTTP 200 with `status: ok` from `/healthz` and HTTP 200
  with `status: ready` from `/readyz` through a temporary local port-forward.
  Ingress was disabled. The workers used the fresh NATS instance; object
  storage pointed at the existing test SeaweedFS service, but no messages were
  available for processing and no object operation was performed.
- The runtime-role-isolation tool passed against the new databases and was
  rerun against the current `myota-test` database URLs: each owner credential
  passed its own database checks, all six cross-owner attempts were denied,
  and no application rows were read. The test deployment was at chart revision
  32 with 21/21 application replicas available after the check.
- Removed the temporary namespace, its PostgreSQL and NATS PVCs, and the
  read-only role-check Job in `myota-test`. The persistent test release and its
  data were not changed.

This completes the fresh-volume initialization, runtime-role isolation, and
API/worker readiness checks for #88. It does not qualify application workflows
on these temporary databases, SMTP delivery from this temporary release,
object-store recovery, or the remaining #10 observability and rollback gates.

## Related deployment state

The persistent Helm release was `myota-test`, chart
`myota-next-1.1.6-testing.492a44b.1` (revision 26), with immutable image
digests and all 15 application Deployments available at the time of this
record. Its Participant registration smoke test is tracked in issues #91 and
#100. Observe reporting remained disabled because `myota-test` had no Observe
credential Secret; see issue #90.
