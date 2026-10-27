# Isolated test deployment

**Current verification:** 2026-10-26 23:58 UTC
**Initial seed verification:** 2026-10-14 11:09 UTC
**Cluster:** local single-node K3s on `spainip-k3s`
**Namespace:** `myota-test`
**Current Fleet source:** `myota-platform/myota-next`, branch `testing`, commit
[`c10df2f`](https://github.com/myota-platform/myota-next/commit/c10df2f22479a676318256c637d90d4f9eb89fd5), path `deploy/fleet/myota-test`.

This is a separate Helm release with its own PostgreSQL, JetStream, SeaweedFS,
and observability storage. It is not a production replica service: the initial
copy is a one-time seed, and production data is not continuously replicated.
The live `myota` namespace and its persistent volumes were left unchanged.

## Current MyOTA Next rollout — 2026-10-26 23:58 UTC

The protected rollout path installed the approved immutable images through
Rancher's catalog operation. This closes the App ownership and recovery
incident described in the historical entries below; it does not complete the
remaining Phase 6 qualification work in
[issue #10](https://github.com/myota-platform/myota-next/issues/10).

- **Fleet source:** `fleet-local/myota-next-test` is Ready at
  `myota-test-deploy`, commit
  [`c10df2f`](https://github.com/myota-platform/myota-next/commit/c10df2f22479a676318256c637d90d4f9eb89fd5).
  `Reconciling=False` and `Stalled=False`.
- **Rancher catalog:** The protected workflow installed chart
  `1.1.6-testing.c10df2f22479.1` as Helm release `myota-test`, revision 32,
  status
  `deployed`. Fleet reports `Ready=True`, `Reconciling=False`, and
  `Stalled=False` at the same commit.
- **Immutable images:** Workflow
  [40420558153](https://github.com/myota-platform/myota-next/actions/runs/40420558153)
  completed the protected Rancher catalog upgrade and verified the release.
  All 15 chart-managed application Deployments are ready. The Activity worker
  runs
  `ghcr.io/myota-platform/myota-next-activity-worker@sha256:eb1e4bc419b2ee459e40734f99ce0465e9a3a824bb2751cc8564516e5e871411`.
- **CI:** Node CI
  [40420558220](https://github.com/myota-platform/myota-next/actions/runs/40420558220)
  passed on merge commit `c10df2f`; this included the OSM PBF decoder setup,
  dependency and source scans, migrations, unit/e2e tests, browser tests, and
  builds. The testing image publication workflow
  [40420558179](https://github.com/myota-platform/myota-next/actions/runs/40420558179)
  also succeeded for the merge; the release change pinned the already
  published Activity worker digest and did not rebuild application images.
- **Public endpoints:**
  `https://myota-admin-test.spainip.es/`,
  `https://myota-participant-test.spainip.es/`, and API
  `https://myota-test.spainip.es/healthz` returned HTTP 200. The API root is
  not a web application and returns 404 by design.
- **Activity worker image publication:** Workflow
  [40419038331](https://github.com/myota-platform/myota-next/actions/runs/40419038331)
  published the worker built from testing commit `8ac4b99`; the protected
  rollout now pins its immutable digest. The digest-only promotion did not
  rebuild application images.
- **Email qualification:** Deployment SMTP success, TLS, failure/retry,
  deduplication, and notification outcomes are recorded under
  [issue #13](https://github.com/myota-platform/myota-next/issues/13), which
  is closed. The test deployment uses its Mailpit sink for synthetic email
  flows.
- **Participant Activity journey:** A signed-in synthetic Participant used
  the browser UI to upload a one-record ADIF file against an open activation.
  Import `e5a1cc51-7542-47f2-989b-fdb55e9ff24b` completed with 1 record seen,
  1 accepted, and 0 rejected. Activity PostgreSQL contains one `VALID` QSO
  linked to that import, and the activation's QSO count is 1. This verifies a
  successful browser-to-worker-to-database import path.
- **ADIF terminal failure:** An earlier synthetic import used a QSO timestamp
  before the activation start, so the fixture was invalid. That failure
  exposed a real worker retry loop: retries were emitted as new JetStream
  messages and the broker delivery count reset. The fix caps attempts using
  durable job state and persists terminal import failure, one dead-letter, and
  one deduplicated failure notification together. In the isolated deployment,
  the invalid job ended `FAILED` at attempt 906, its import ended `FAILED` with
  one rejected record, and one dead-letter plus one failure notification were
  present. A 10-second follow-up query showed no additional attempts. The
  separate valid import above passed afterward.
- **JetStream restart and redelivery:** A temporary file-backed stream held an
  unacknowledged probe message while the single `myota-test-nats-0` pod was
  deleted and recreated by its StatefulSet. The client reconnected and received
  delivery 2 for the same message ID, acknowledged it, then observed zero
  acknowledged-pending and zero queued messages. The probe deleted its durable
  consumer and stream afterward. The replacement NATS pod returned to Ready
  with its existing persistent volume.
- **Rollback rehearsal:** Catalog chart
  `1.1.6-testing.0e4eb202bda2.1` was restored as revision 30, then chart
  `1.1.6-testing.6fc64541f37f.1` was re-applied as revision 31 by workflow
  [40412857182](https://github.com/myota-platform/myota-next/actions/runs/40412857182).
  All 16 namespace Deployments were ready after restoration and the three
  public endpoints above returned HTTP 200. Revisions 29 and 30 had identical
  release values and all 15 rendered application Deployment images, so this
  only verifies the catalog/Helm rollback path. It does not verify compatibility
  across changed code, image digests, schemas, or persisted NATS work.

The earlier failed workflow after PR #131 did not start a catalog operation and
did not modify the then-current Helm release or persistent volumes. PR #132
changed the protected workflow to permit a missing App metadata baseline, then
require Rancher to recreate it, advance the chart revision, and verify every
approved image digest. PR #132 passed Node CI and all 18 image build/scan
checks before merging to `testing`; the successful live run above proves the
recovery path.

### Remaining Phase 6 gates

- [Issue #88](https://github.com/myota-platform/myota-next/issues/88): a
  disposable namespace with new 10/20/50 GiB PVCs was initialized through the
  repository infrastructure and migration charts. The migration hook applied
  21/5/7 Core/Activity/Geodata migrations and repeated them successfully on a
  second Helm upgrade. The runtime-role isolation Job verified owner-only
  access without reading rows. A later isolated Helm release started all API
  and browser Deployments plus Activity/Geodata/outbox/email workers against
  fresh PVCs: all 15 Deployments and 20 application replicas were available;
  all 3 PostgreSQL pods and the isolated JetStream pod were healthy. The
  provisioner validated four streams and eleven durables. Gateway `/healthz`
  and `/readyz` returned HTTP 200. The runtime-role-isolation tool also passed
  against the current `myota-test` database URLs, denying all six cross-owner
  credential attempts without reading application rows. Ingress was disabled,
  SMTP delivery was not exercised in this scratch release, and the optional
  Geodata source importer remained disabled. Both temporary namespaces and
  their PVCs were removed. Full procedure:
  [fresh-database evidence](evidence/phase6-fresh-database-qualification-2026-10-26.md).
- [Issue #89](https://github.com/myota-platform/myota-next/issues/89): database
  and object-store snapshots were restored and integrity-checked. The
  repository PostgreSQL backup/restore wrapper also passed against newly
  created disposable databases on each isolated owner instance using
  PostgreSQL 16 tools: it created checksummed custom-format dumps, rejected an
  incorrect restore confirmation, restored mutated marker values, and removed
  its probe tables, backup directory, and scratch databases. This exercises the
  wrappers against the test PostgreSQL endpoints. The object-store utility
  also restored one synthetic 51-byte object and verified its content, hash,
  metadata and tag. These checks exercise the tool paths, but the earlier source
  snapshot was not representative and writers were not quiesced. Off-node
  retention, measured RTO, full-data consistency, and PVC-loss recovery remain
  unqualified.
- [Issue #90](https://github.com/myota-platform/myota-next/issues/90):
  `@nestjs/observe` dependencies, root hooks, stable service IDs, and the
  no-credentials startup path are verified. Authenticated Observe credentials
  and delivery to an Observe destination are not configured or verified.
- [Issue #92](https://github.com/myota-platform/myota-next/issues/92): the
  single-instance dependency restart paths are verified. JetStream redelivered
  an unacknowledged message after pod recreation; the Activity worker's durable
  attempt cap, terminal import failure, single dead-letter, and deduplicated
  notification were verified alongside a successful ADIF import. HA and
  multi-instance recovery are outside this Phase 6 scope.
- [Issue #91](https://github.com/myota-platform/myota-next/issues/91): on
  revision 29, Participant sign-in, sign-out, profile persistence across a
  fresh session, registration/email verification through Mailpit, and base
  `PARTICIPANT` role assignment passed. A `PARTICIPANT_ADMIN` browser session
  searched for a Participant, suspended and reactivated sign-in, and received
  API 403 plus UI denial for broader Identity administration. The temporary
  support account was anonymized, the test email was deleted, and the target
  account was restored to `ACTIVE`. On revision 32, a Participant created and
  used an activation, and the browser ADIF workflow completed with a persisted
  valid QSO; see the evidence above. Correlated telemetry and audit evidence
  remain unqualified, as do additional activation/QSO edit and closure cases.
- [Issue #93](https://github.com/myota-platform/myota-next/issues/93): the
  catalog/Helm rollback path is exercised, but only between releases with the
  same values and image digests. Compatibility across changed code, image
  digests, schemas, and persisted database, object-store, or NATS work remains
  unqualified.

These are qualification gaps, not evidence that the deployment failed. Keep
issue #10 open until its exit criteria pass or each residual limitation has an
explicit accepted disposition and the required evidence is linked.

## Historical MyOTA Next rollout — 2026-10-26 20:31 UTC (superseded)

At this time the Fleet values and deployed Helm release had diverged. Fleet
was Ready at an earlier `myota-test-deploy` commit while Helm revision 26 was
still serving chart `myota-next-1.1.6-testing.492a44b.1`. Updating the values
bundle did not create a Helm revision. The protected catalog workflow
introduced afterward addresses this limitation; see the current rollout above
and [issue #118](https://github.com/myota-platform/myota-next/issues/118).

The isolated GitHub Actions environment restricts catalog upgrades to the
`testing` branch and does not allow administrator bypass. Credentials remain
in the ignored, mode-600 local `runtime-secrets` file and Kubernetes secret
store, not in Git. The successful protected workflow is the source of current
release evidence; earlier PR #124 review, App conflict, and pre-rollout digest
descriptions are retained only as incident history.

#### Earlier database and runtime-role rehearsal — 2026-10-26 20:31 UTC

Three uniquely named databases were confirmed absent, created on the isolated
Core, Activity, and Geodata PostgreSQL instances, then initialized with the
chart migration Job. The first run applied 21 Core, five Activity, and seven
Geodata migrations; a second run reported all schemas current. A concurrent
migration Job waited for a held Core advisory lock and completed after release.
The scratch databases, Jobs, and temporary migration URL Secret were removed.

The same Job reported the three existing `myota-test` schemas current and
synchronized owner-specific runtime roles. Read-only checks authenticated to
each database with its owning role and denied all six cross-owner attempts.
These checks support, but do not replace, the clean newly provisioned storage
installation evidence required by #88.

## Historical monorepo rollout — 2026-10-14 21:43 UTC

| Check | Result |
|---|---|
| Fleet | The Testing-only GitRepo `fleet-local/myota-deploy-testing` reads `myota-platform/myota`, branch `testing`, path `deploy/helm/myota-test`, at commit [`1140d05`](https://github.com/myota-platform/myota/commit/1140d057ca9122665414dede665c1677536a6941). At 21:43 UTC its display state was `WaitApplied` (`0/1` ready BundleDeployments), although its resource counts were 76/76 ready with zero missing, modified, or not-ready resources. The BundleDeployment's last Ready condition was true at 21:38 UTC. Production Fleet remains on `myota-platform/myota-deploy`, branch `main`. |
| Helm and storage | Helm release `myota-test` is at revision 327 with status `pending-upgrade` (`Preparing upgrade`). All 10 database and observability PVCs remain Bound. Revision-327 migration, JetStream, Activity notification, and PostgreSQL monitor-role Jobs completed. |
| Workloads | All 24 Deployments and all 3 StatefulSets report their desired replicas ready. The Fleet display and Helm release have not yet returned to a settled state. |
| Profile storage | `myota-test-user-profile-images` uses the dedicated Testing credential and static S3 policy. Authorized write/read/delete passed; access to `myota-adif` and anonymous bucket listing were denied. The temporary object and Jobs were removed. Production was not changed. |
| MyOTA service health dashboard | Grafana's `myota-observability` ConfigMap and mounted dashboard file contain `myota-service-health.json` with title **MyOTA service health** and UID `myota-service-health`. It is provisioned in the **MyOTA** folder and uses the `MyOTA Prometheus` datasource (`myota-prometheus`). Live Prometheus queries return `myota_identity_users` values `OPERATOR=2` and `SWL=0`, and `myota_programmes=2`; these match the corrected dashboard expressions. The old `myota-operations` dashboard was retired and is not the current dashboard. |
| Logs dashboard | The deployed `myota-observability` ConfigMap contains `myota-logs-failures.json` with UID `myota-logs-failures`; Grafana logged `finished to provision dashboards` at 19:42 UTC. It uses the MyOTA Loki datasource. |
| Trace dashboard | The mounted `myota-traces.json` has UID `myota-traces`, finite service options, and the `All` choice from merged PR [#80](https://github.com/myota-platform/myota/pull/80). Its SHA-256 is `510f978c3a78a571b4082dc34894167bd5da97993ddf78b1425ad9236fb8de8d`. |
| Logs dashboard | The mounted `myota-logs-failures.json` now contains the bounded filters from PR [#81](https://github.com/myota-platform/myota/pull/81), with SHA-256 `a3aec40e56381be3a746484d50ce3dfd52d1342c7b6592d02dade517a270171c`. Its mounted bytes match the `myota-observability` ConfigMap. Grafana logged successful dashboard provisioning at 21:30:37 UTC. |
| Existing telemetry | A read-only Tempo search returned up to 1,000 current traces and nine distinct MyOTA root service names. The failed-trace and over-one-second TraceQL queries parsed successfully and returned zero matches in the queried range; no synthetic traces were inserted. |
| Digest workflow | The active workflow is `.github/workflows/sync-testing-image-digests.yml` in `myota` and runs on GitHub-hosted Actions runners. The old `.github` workflow is deleted. Latest manual run [#38634904281](https://github.com/myota-platform/myota/actions/runs/38634904281) succeeded but skipped token creation and digest resolution because the MyOTA-scoped GitHub App credentials are absent in the `myota` Actions context. The workflow is not a Kubernetes CronJob. See [myota issue #30](https://github.com/myota-platform/myota/issues/30) for setup and [issue #53](https://github.com/myota-platform/myota/issues/53) for image publication and Fleet evidence. |
| Profile images | PR #78 recorded the Testing image digests. Identity runs `sha256:7cbecb4d64db98300312237e5d776d8f8ae5ab3126a6cbe8fada58e1f77ddfdf`; Participant Web runs `sha256:5a404d56b721532c09b60670de410653fa551cebb20d9b0afe72c8fad6542249`. |

The Logs / Failures and Distributed Traces dashboards are provisioned in
Grafana's **MyOTA** folder. Both default to a 30-minute time window and a
30-second refresh. The trace dashboard has the finite service dropdown from PR
#80, and the Logs / Failures dashboard has the bounded filters from PR #81.
The configured Loki and Tempo datasource UIDs are `myota-loki` and
`myota-tempo`. The current Tempo search found real traces but no failed or
over-one-second matches; no synthetic traces were inserted. Fleet/Helm
reconciliation and authenticated interactive navigation remain open
acceptance gates under [myota #22](https://github.com/myota-platform/myota/issues/22),
[myota #48](https://github.com/myota-platform/myota/issues/48), and
[myota #53](https://github.com/myota-platform/myota/issues/53).

The service-health dashboard was corrected in monorepo PR
[#75](https://github.com/myota-platform/myota/pull/75) after a live query audit
found that Identity and Programme export `myota_identity_users` and
`myota_programmes`, without the `_total` suffix used by the old panels. Both
the Compose source and packaged Helm dashboard now use the observed names.
Focused observability tests and the PR checks passed before merge to
`testing`; Fleet applied commit `5b334661` at Helm revision 304. The dashboard
payload in the live ConfigMap has SHA-256
`d3a1a3d3270b5458b9369d8b0092ed68e23fddc481fba065bc3329dd8b52b311`.
The mounted file has SHA-256
`39ccd622a714b3e9ab75fbbd5ecfc35f95068b32126e63a69f8a08205ba95308`; the
ConfigMap value has a trailing newline absent from the mounted file. Both
contain the same dashboard UID and corrected Identity and Programme queries.
The Prometheus results are current Testing values, not target or production
counts. Grafana's dashboard provisioning is verified from its ConfigMap and
provisioning logs; authenticated browser navigation remains a separate
acceptance gate tracked in [issue #48](https://github.com/myota-platform/myota/issues/48).

To enable digest updates, install the digest-writer GitHub App on `myota` with Contents and Pull requests write access. Add `TESTING_DIGEST_APP_ID` as an Actions variable and `TESTING_DIGEST_APP_PRIVATE_KEY` as an Actions secret in that repository, then run the workflow again. See [digest writer setup](https://github.com/myota-platform/myota/issues/30) and [image access and rollout evidence](https://github.com/myota-platform/myota/issues/53). The latest digest workflow run confirms that those credentials are currently missing.

## Historical deployment snapshot — 2026-10-14 11:09 UTC

| Check | Result |
|---|---|
| Helm | Release `myota-test`, chart `myota-test-0.1.0` with shared chart dependency `0.2.25`, revision 191, `deployed`. |
| Fleet | GitRepo tracks Deploy `testing` at `3af93b275879c60cef13426fb3ac483f76ac996d`; Fleet reports 76/76 desired resources ready and BundleDeployment Ready. |
| Workloads | All 24 Deployments and all 3 StatefulSets have their desired replicas available. Revision 191 migration, JetStream provisioner, Activity notification provisioner, and PostgreSQL monitor-role Jobs completed successfully. |
| Images | Pinned testing digests are active. Geodata API and import worker use the digest published from Geodata `testing` PR #26 and recorded through Deploy PR #59. |
| JetStream | The deployment-owned provisioner validated all three streams and ten selected work durables. |
| Admin UI | [admin.test.myota.top](https://admin.test.myota.top) returns HTTP 200 with a valid ACME certificate. |
| Participant UI | [test.myota.top](https://test.myota.top) returns HTTP 200 with a valid ACME certificate. |
| API ingress | `api.test.myota.top` terminates with a valid ACME certificate; `/` returns 404 because the gateway does not serve a root route. |

Fleet watches the Deploy `testing` branch and corrects drift. The test values
pin immutable image digests, so a digest update rolls the corresponding
workload. Deploy PR [#49](https://github.com/myota-platform/myota-deploy/pull/49)
updated four image digests, and Fleet deployed Helm revision 35. Those
historical values were:

| Image | Testing digest |
|---|---|
| Geodata service (superseded on 14 October) | `sha256:d0c99a7aa47cf876ab6f4b7b8add65b0e37a48f387d840e3c7e46ff7b6cef36a` |
| Gateway | `sha256:5ea1e5b82eca775b855c9699904a6fefe78ee39dfa10398da68449d4f8e3ce94` |
| Admin Web | `sha256:cdbcc11c0fdbdb6dfdc94d6997343d9ece310946edc537b4faa2f4ac16863394` |
| Shared Platform image | `sha256:3abc60cd6d276afd1d28ce765e40920669d00daa284a317cb4787b78ded36e64` |

Earlier on 14 October, Geodata PR [#23](https://github.com/myota-platform/myota-geodata-service/pull/23)
merged to `testing`. Deploy PR
[#58](https://github.com/myota-platform/myota-deploy/pull/58) recorded its
published digest, `sha256:929b31b9f9509c96ed088e158dc3c14e70c95b6b20ab510ca093b861edbb14ba`.
Fleet deployed it to both Geodata Deployments at Helm revision 176; those
Deployments and the other workloads were ready, and `/healthz` returned
HTTP 200.

| Current image | Testing digest |
|---|---|
| Identity service | `sha256:d1b35ba5338d50f3b77627da99a9ca170735ff461c014bd4b105ba098469de17` |
| Programme service | `sha256:3cd2da5d6afe945ad505bdce8fb867789cc7a9076798dce52818a45745f43fc0` |
| Geodata service | `sha256:c6c8ac57d4590f19bf990ca21282bd1084be0474d3014f094a2a51475ef0eaaf` |
| Activity service | `sha256:653964a8c74cf485421449a122dd1397fc60f6a75efa2af90d55c4d094a22136` |
| Gateway | `sha256:5ea1e5b82eca775b855c9699904a6fefe78ee39dfa10398da68449d4f8e3ce94` |
| Admin Web | `sha256:8062833ca0052a2cb53b085517c79a7a4a1f560f0c1cc736f45a2e8b29038164` |
| Shared platform image | `sha256:3abc60cd6d276afd3d28ce765e40920669d00daa284a317cb4787b78ded36e64` |
| Participant Web | `sha256:563f0215e675b2a5824da264c6af8d852720192fdbe6fd951b4f6ffcc9cf98ec` |

Later on 14 October, Geodata PR
[#26](https://github.com/myota-platform/myota-geodata-service/pull/26)
merged with catalogue-only duplicate comparison. The digest workflow opened
Deploy PR [#59](https://github.com/myota-platform/myota-deploy/pull/59), which
passed runtime tests, chart rendering, Python quality, and both artifact build
checks before merging. Fleet deployed the current Geodata digest to the API
and import-processing worker at Helm revision 191. Both Deployments are
ready, the test bundle reports 76/76 desired resources ready, and Geodata
`/healthz` returned HTTP 200. A read-only call to the deployed duplicate
lookup returned two nearby `APPROVED` catalogue matches and no other lifecycle
statuses; it wrote no test records.

After PR #49, all 24 Deployments were available and both test UIs returned HTTP
200. Deploy PR [#50](https://github.com/myota-platform/myota-deploy/pull/50)
then removed the ineffective duplicate digest poller from the `testing` branch
and updated its operator README. It merged as
[`e4791b4`](https://github.com/myota-platform/myota-deploy/commit/e4791b4b1dd13fd4bd4d3c2672c2d42cdd73ac56).
That commit contains only the workflow deletion and README update. Fleet
reconciled it as Helm revision 37; the release is `deployed`, the BundleDeployment
is Ready, all 27 workload controllers have desired replicas available, the
revision 37 setup Jobs completed, and both UI endpoints return HTTP 200.

### Rollout recovery

Changing the service image digest exposed an immutable legacy JetStream Job.
The chart was changed to use revision-scoped Jobs, Fleet's cached shared chart
dependency was advanced to `0.2.24`, and first-install stream and durable
provisioning was moved to retryable, revision-scoped Jobs in `0.2.25` (Deploy
PRs [#41](https://github.com/myota-platform/myota-deploy/pull/41),
[#42](https://github.com/myota-platform/myota-deploy/pull/42), and
[#43](https://github.com/myota-platform/myota-deploy/pull/43)). The isolated
Helm release history then became inconsistent, so the test release was
reinstalled from Fleet. All three database PVCs were preserved. The
chart-managed SeaweedFS volume was recreated; its live source remained intact,
and the existing application bucket was copied back and independently
verified. Production workloads and volumes were not changed.

### Image update automation status

The digest-sync workflow now lives in the MyOTA monorepo at
[`.github/workflows/sync-testing-image-digests.yml`](https://github.com/myota-platform/myota/blob/main/.github/workflows/sync-testing-image-digests.yml).
It runs on a five-minute schedule or manual dispatch, resolves published
`testing` tags to immutable digests, and opens a digest-only PR against
`myota/testing`. The old copy was removed from the organization
[`.github`](https://github.com/myota-platform/.github) repository after the
monorepo workflow reached `main`.

The workflow expects `TESTING_DIGEST_APP_ID` and
`TESTING_DIGEST_APP_PRIVATE_KEY` in the `myota` Actions context (or
organization-level settings that include `myota`). These values are
not readable through the GitHub connector. The first scheduled run after the
move has not yet been observed, so credential availability and digest PR
creation remain unverified. The workflow includes `workflow_dispatch` for
a manual verification run. The test digest values and live `myota-test`
workloads still reflect the previous image set; Fleet rollout verification
remains outstanding. Production values were not changed.
## Initial data seed

The database and object data were copied after migrations and before enabling
public test ingress. The first pass missed Identity accounts because it
incorrectly treated Core `public.account` as the Identity source of truth.
Identity stores accounts in `public.service_state.state->'items'` for service
`identity`. After the user explicitly authorized the copy, the requested active
account, password hash, and its active `GLOBAL_OPERATOR` role assignment were
copied into the test Identity state. The test-only failed-attempt counter and
lock state were reset. The test Identity pod was restarted to load the updated
state. The password hash equality, active account status, role assignment, and
running Identity account metrics were verified without printing credential
material. The test login was not exercised because the user's password was not
provided to this session.

The copy excluded other production accounts, all sessions and refresh-token
state, recovery tokens, revoked-token entries, login-rate-limit history, and
production security/audit history. The test deployment continues to use its
separate bootstrap account.

Verified source/target data counts were:

| Data | Source and test result |
|---|---:|
| Identity `service_state.state.items` | 2 live accounts; requested account copied; 1 copied account persisted in test |
| Identity password credential | Requested account's password hash matches; test lockout counters reset |
| Active Identity role assignment | Requested account's `GLOBAL_OPERATOR` role copied and verified |
| Legacy Core `public.account` relation | 0 rows; not the Identity source of truth |
| Activity `activity_notification` | 155 rows |
| Geo `geodata_entity` | 10 rows |
| Geo `geodata_import_candidate` | 10 rows |
| Geo `source_reference` | 10 rows |
| Geo `geodata_audit_event` | 9,800 rows |
| Target migration-owned `geodata_schema_feature` marker | 1 row retained |

The seed scripts recorded these per-database completion markers in
`myota-test`:

- `myota.dev/live-data-seed-myota_core=complete`
- `myota.dev/live-data-seed-myota_activity=complete`
- `myota.dev/live-data-seed-myota_geo=complete`

Target migration ledgers, seeded spatial reference data, `entity_type`, and
the target-owned `geodata_schema_feature` marker were preserved. Four named
application buckets were absent in the live SeaweedFS. The existing
`myota-geodata-imports` bucket was copied and independently checked by object
key and size: 10 matching objects totaling 13,625,617 bytes. After the
isolated release recovery, the same source bucket was copied again and
reverified with the same 10 object keys, sizes, and total byte count.

Outbox delivery state, dead-letter and replay state, consumer checkpoints,
sessions, recovery tokens, observability history, and live JetStream messages
were not copied. Test JetStream starts independently with the provisioned
streams and durable consumers; production messages are not replayed into it.

Temporary live SeaweedFS source credentials and the object verifier Job and
ConfigMaps were removed after the copy check. Test secrets remain local to the
K3s namespace and their source files are stored under the host's
`runtime-secrets` directory; secret values are not recorded here.

## Ownership and operating notes

- The current Testing Helm source and Fleet bundle are in the monorepo at
  [`deploy/helm/myota-test`](https://github.com/myota-platform/myota/tree/testing/deploy/helm/myota-test). The old `myota-deploy` repository remains the production Fleet source until that environment is deliberately migrated.
- The test release uses `values-testing.yaml` and
  `values-image-digests-testing.yaml`; production values remain under
  `deploy/helm/myota`.
- Re-run the one-time seed only after reviewing the per-database markers and
  target rows. The object copy is safe to repeat because it overwrites matching
  keys, but it does not delete target-only objects.
- This snapshot proves readiness and endpoint responses at the stated time.
  It does not qualify single-node failure recovery, production capacity, or
  continuous data synchronization.
