# Phase 6 fresh-database qualification evidence — 2026-10-26

This record covers migration and service-readiness checks for disposable databases
in the isolated `myota-test` PostgreSQL instances. It does not claim that a new
PostgreSQL PVC was initialized end to end.

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

## Limitation and follow-up

The disposable databases were created inside already initialized PostgreSQL
instances. The PostgreSQL init container's `020-create-app-role.sh` runs only
for the initial database on a new data volume, so its per-database default
privileges were not applied to these later-created scratch databases. Identity
initially failed with a permission error for `myota_app`. For this scratch test
only, grants and default privileges were added to the three disposable databases;
Identity then became Ready. The manual grants and all scratch databases were
removed afterward.

This setup is not equivalent to a new PostgreSQL volume. Before closing #88,
repeat the migration and readiness check using newly initialized test database
volumes, allowing the normal PostgreSQL role-init script to create the app role
and default grants. Keep the test release isolated and preserve all existing
`myota-test` PVCs.

## Related deployment state

The persistent Helm release was `myota-test`, chart
`myota-next-1.1.6-testing.492a44b.1` (revision 26), with immutable image
digests and all 15 application Deployments available at the time of this
record. Its Participant registration smoke test is tracked in issues #91 and
#100. Observe reporting remained disabled because `myota-test` had no Observe
credential Secret; see issue #90.
