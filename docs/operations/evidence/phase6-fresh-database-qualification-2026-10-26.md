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

- Recreated a temporary namespace with new Core, Activity, and Geodata PVCs
  and deployed the current `myota-test` immutable application image digests
  through the repository Helm chart. Migration hooks initialized each database
  from empty storage; the migration ledgers contained 21 Core, 5 Activity, and
  7 Geodata migrations.
- Started the Gateway, Identity, Identity maintenance, Programmes, Activity,
  Geodata, Admin web, and Participant web workloads. All 8 Deployments had all
  13 desired application replicas available, and all 3 PostgreSQL pods were
  ready (16/16 pods total), with no restarts at verification time. The
  isolated Gateway returned HTTP
  200 with `status: ok` from `/healthz` and HTTP 200 with `status: ready` from
  `/readyz` through a temporary local port-forward.
- Ingress was disabled. Activity and Geodata workers, notification delivery,
  email delivery, and the three outbox workers were disabled, so this did not
  verify NATS-backed work consumption or worker readiness on the fresh
  databases. The first chart revision briefly created outbox pods without the
  NATS Secret; the corrected second revision disabled those workers and
  reached the ready state recorded above. The temporary namespace, Helm
  releases, PVCs, and port-forward were removed. The persistent `myota-test`
  release and its data were untouched.

This closes only the fresh-database API and browser readiness portion. Keep
the worker-readiness task in #88 open until the NATS-backed workers have been
verified against fresh storage.

## Related deployment state

The persistent Helm release was `myota-test`, chart
`myota-next-1.1.6-testing.492a44b.1` (revision 26), with immutable image
digests and all 15 application Deployments available at the time of this
record. Its Participant registration smoke test is tracked in issues #91 and
#100. Observe reporting remained disabled because `myota-test` had no Observe
credential Secret; see issue #90.
