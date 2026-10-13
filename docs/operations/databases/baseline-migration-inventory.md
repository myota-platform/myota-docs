# Database migration baselines and data-preserving rebuilds

**Status verified:** 13 October 2026 09:15 UTC  
**Scope:** Deploy, Activity, Geodata, Platform, and the local non-production K3s
cluster. This record covers baseline source and disposable runner evidence. It
does not authorize or report a live database rebuild.

## Ownership and synchronized copies

| Database | Schema owner and authoritative baseline | Deployment runner and mirror | Status |
|---|---|---|---|
| `myota_core` | Deploy: [`db/migrations/core/001_core_baseline.sql`](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/core/001_core_baseline.sql); shared by Identity and Programme | Deploy owns [`db/migrations/run.sh`](https://github.com/myota-platform/myota-deploy/blob/testing/db/migrations/run.sh). Platform mirrors runner and SQL at [`db/migrations/core/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/core). | Fresh and repeat runner verified in disposable K3s DB; current Helm image is still on historical migration chain. |
| `myota_activity` | Activity: [`migrations/001_activity_baseline.sql`](https://github.com/myota-platform/myota-activity-service/blob/testing/migrations/001_activity_baseline.sql) | Deploy mirrors at [`db/migrations/activity/`](https://github.com/myota-platform/myota-deploy/tree/testing/db/migrations/activity); Platform mirrors at [`db/migrations/activity/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/activity). | Fresh and repeat runner verified in disposable K3s DB; current Helm image is still on historical migration chain. |
| `myota_geo` | Geodata: [`migrations/001_geodata_baseline.sql`](https://github.com/myota-platform/myota-geodata-service/blob/testing/migrations/001_geodata_baseline.sql) | Deploy mirrors at [`db/migrations/geo/`](https://github.com/myota-platform/myota-deploy/tree/testing/db/migrations/geo); Platform mirrors at [`db/migrations/geo/`](https://github.com/myota-platform/myota-platform/tree/testing/db/migrations/geo). | Fresh and repeat runner verified against PostGIS 3.4.3 in disposable K3s DB; current Helm image is still on historical migration chain. |

The owner repositories define schema content. Deploy owns the combined database
migration runner and deployment wiring. Platform is a synchronized mirror and
must not become an independent schema authority.

## Baseline and runner evidence

The three baseline source PRs and mirror PRs are merged to `testing`:

- Core/runner: [Deploy PR #18](https://github.com/myota-platform/myota-deploy/pull/18)
  and [Platform PR #3](https://github.com/myota-platform/myota-platform/pull/3).
- Activity: [Activity PR #7](https://github.com/myota-platform/myota-activity-service/pull/7).
- Geodata: [Geodata PR #11](https://github.com/myota-platform/myota-geodata-service/pull/11).

An end-to-end runner rehearsal exposed PostgreSQL psql compatibility failure:
the 16.4 client rejected pg_dump's `\\restrict` and `\\unrestrict`
metacommands embedded in the Core and Activity baseline files. Those lines were
removed from the authoritative/mirror SQL and the compatibility fix was merged
to `testing` in [Deploy PR #19](https://github.com/myota-platform/myota-deploy/pull/19),
[Activity PR #8](https://github.com/myota-platform/myota-activity-service/pull/8),
and [Platform PR #4](https://github.com/myota-platform/myota-platform/pull/4).
The SQL bodies remain byte-identical across their owners and mirrors.

The fixed runner and migration SQL were tested in the isolated
`codex-db-rebuild` namespace using the PostgreSQL 16/PostGIS test pods:

- Fresh install created exactly one `baseline-v1` ledger row per database and
  set all three readiness rows to `ready=true`.
- A repeat run skipped all recorded baselines, retained one ledger row each,
  and remained ready.
- Failure injection hid Core's `public.account` sentinel. The runner failed
  with the expected error and left all three readiness rows false.
- Restoring the sentinel and expected baseline checksum let the runner
  complete and restore readiness across all three databases.
- Runner jobs, ConfigMap, and the three runner-test databases were deleted after
  evidence capture. No live MyOTA database was changed by these checks.

CI evidence for the baseline changes includes Activity run
[38368130423](https://github.com/myota-platform/myota-activity-service/actions/runs/38368130423),
Geodata runs
[38370613580](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370613580)
and
[38370614459](https://github.com/myota-platform/myota-geodata-service/actions/runs/38370614459),
Deploy runs
[38370753170](https://github.com/myota-platform/myota-deploy/actions/runs/38370753170)
and
[38370752130](https://github.com/myota-platform/myota-deploy/actions/runs/38370752130),
Platform run
[38370757925](https://github.com/myota-platform/myota-platform/actions/runs/38370757925).
The compatibility-fix PR runs also passed: Deploy
[38374441554](https://github.com/myota-platform/myota-deploy/actions/runs/38374441554),
Activity
[38374446912](https://github.com/myota-platform/myota-activity-service/actions/runs/38374446912),
and Platform
[38374443683](https://github.com/myota-platform/myota-platform/actions/runs/38374443683).

These checks establish baseline SQL and runner behavior in a disposable
environment. They do not establish Identity/Programme or Activity API/worker
integration acceptance, deployment of the fixed migration image, or safe live
cutover.

## Live runtime boundary

Helm release `myota` is revision 282, created as a rollback to revision 281.
MyOTA Fleet still tracks Deploy `main` commit `9d80741`. The completed live
migration Job `myota-migrations-281` executed the historical numbered chain,
including Geodata migration 022, and completed successfully. It did not execute
the new baseline runner from `testing`.

All live databases remain intact and their readiness markers remain healthy.
No go/no-go has been recorded. Do not drop or recreate a live database until
the fixed runner is built and deployed, baseline issue acceptance is complete,
writers and asynchronous work are quiesced, and a final checksummed archive has
passed isolated restore validation.

## Recovery archive evidence

The following custom-format archives were captured while services were still
active. Each passed archive inspection and an isolated restore. They are
**preliminary rehearsal archives only**, not cutover backups:

| Database | Size | SHA-256 | Isolated restore evidence |
|---|---:|---|---|
| Core | 112,415 bytes | `420822e5185845c98b42ba226e0eeac6917db23c70d091eb77a7415e94c41c64` | 29 public tables restored. |
| Activity | 863,447 bytes | `dc1901822aa198e024e0a1f737e6a89845188df1c6a5ceb9e4b688559eeb4ac` | 26 public tables restored. |
| Geodata | 9,647,821 bytes | `89a321ff6ae221ba4ae1cda3f3cc3e221b3ff9840ea4a9a3153a52b438642034` | 28 public tables restored; PostGIS available; normalized schema matched the fresh baseline. |

Because capture happened before quiescence, these archives do not satisfy the
cutover gate. New archives must be taken after service writers, scheduled work,
outbox relays, and JetStream consumers are paused or fenced and in-flight work
is accounted for. Restore only data into the verified schema; exclude baseline
bookkeeping (`myota_schema_migration` and
`myota_deployment_schema_state`). For Geodata, also exclude
`geodata_schema_feature` so the baseline-owned `row_authority_v1` marker
survives. Preserve sequences, large objects, domain rows, outbox state,
idempotency, and consumer checkpoints.

The rehearsal namespace retains its restore pods and archived databases until
the paired recovery evidence is complete. Remove it after evidence capture and
remove the restricted local archive directory after acceptance.

## Issue status and remaining gates

All six issues remain open:

| Owner | Baseline issue | Rebuild issue |
|---|---|---|
| Deploy/Core | [#15](https://github.com/myota-platform/myota-deploy/issues/15) — baseline | [#16](https://github.com/myota-platform/myota-deploy/issues/16) — backup and rebuild |
| Activity | [#4](https://github.com/myota-platform/myota-activity-service/issues/4) — baseline | [#5](https://github.com/myota-platform/myota-activity-service/issues/5) — backup and rebuild |
| Geodata | [#8](https://github.com/myota-platform/myota-geodata-service/issues/8) — baseline | [#9](https://github.com/myota-platform/myota-geodata-service/issues/9) — backup and rebuild |

Verified baseline checkboxes and evidence are recorded in those issues. Remaining
baseline gates are exact historical migration/object ownership maps and focused
service integration checks (Identity/Programme for Core; API, worker, outbox,
import/recovery paths for Activity and Geodata). The runner is not yet deployed
from the fixed source in Helm.

Rebuild gates remain: deploy the verified runner image; quiesce writers,
workers, scheduled jobs, relays, and consumers; take and rehearse each final
post-quiescence backup; record per-database go/no-go; recreate one database at a
time from its baseline; data-only restore; validate counts, sequences,
extensions, constraints, read/write paths, outbox/idempotency/checkpoints,
readiness, and recovery; then resume services and clean temporary artifacts.
