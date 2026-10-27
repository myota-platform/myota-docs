# Isolated test deployment

**Current verification:** 2026-10-27 04:07 UTC
**Initial seed verification:** 2026-10-14 11:09 UTC
**Cluster:** local single-node K3s on `spainip-k3s`
**Namespace:** `myota-test`
**Current Fleet source:** `myota-platform/myota-next`, branch `testing`, commit
[`5ed913c`](https://github.com/myota-platform/myota-next/commit/5ed913cee4117433e3aa149edc9da64b21f10fa9), path `deploy/fleet/myota-test`.

This is a separate Helm release with its own PostgreSQL, JetStream, SeaweedFS,
and observability storage. It is not a production replica service: the initial
copy is a one-time seed, and production data is not continuously replicated.
The live `myota` namespace and its persistent volumes were left unchanged.

## MyOTA Next deployment and authenticated qualification — 2026-10-27 04:07 UTC

### Deployment and health

- Node CI [run 40445095599](https://github.com/myota-platform/myota-next/actions/runs/40445095599), changed-image publication [run 40445095579](https://github.com/myota-platform/myota-next/actions/runs/40445095579), and Rancher catalog upgrade [run 40445095573](https://github.com/myota-platform/myota-next/actions/runs/40445095573) completed successfully for `testing` commit `5ed913c`.
- Fleet GitRepo `myota-next-test` reports commit `5ed913cee4117433e3aa149edc9da64b21f10fa9` and 1/1 ready bundle. Helm release `myota-test` is deployed at revision 38, chart `myota-next-1.1.6-testing.5ed913cee411.1`.
- All 16 Deployments have their desired replicas ready. Admin and Participant web roots both return HTTPS 200.

### Metrics and alerts

- In-cluster Prometheus reports all 15 `myota-test` scrape targets `up=1` and no active alerts for the namespace.
- After the ADIF run, the Gateway request metric has 23 series with observed statuses 200, 201, 202, and 403, and no 5xx series. It includes `POST /v1/adif/` with status 202. This is route-level evidence, not trace-ID correlation, and does not prove Observe delivery.
- The authenticated Admin Grafana datasource API returned HTTP 200 and listed only Prometheus and Alertmanager. No Loki/Tempo datasource or matching in-cluster pod was found, so logs and traces are not currently inspectable through this deployment's Grafana view.

### Admin workflows

- An authenticated browser session received HTTP 200 from login and current-session APIs, returned the `GLOBAL_OPERATOR` role, and reached `/dashboard`. In-app navigation opened Participant accounts and Awards workspaces with no API responses at 400 or higher.
- The Awards workspace generated a mock preview for an existing award. The preview API returned HTTP 200 and an `application/pdf` response of 15,397,223 bytes; a browser window opened and the UI confirmed that no award was saved or issued. The test catalogue currently has one `APPROVED` award and no `PUBLISHED` definition, so published-layout editing cannot be qualified without adding persistent test data. Publishing and issuance also remain untested after this image update.

### Participant workflows

- A browser session received HTTP 200 from login and current-session APIs, showed only the `PARTICIPANT` role, and loaded the public entity catalogue and the account's activation history. Credentials and personal email were not recorded.
- Using the synthetic Phase 6 entity, manual activation creation returned 201, QSO creation returned 201, and close returned 200. The closed activation remained visible after sign-out and fresh sign-in.
- ADIF success and terminal-failure paths were recorded on earlier revisions. Requalification after the award-related Activity image update succeeded: a fresh synthetic activation returned 201, ADIF upload returned 202, and authenticated status polling reached `COMPLETED` with 1 accepted and 0 rejected. Close returned 200, and the closed activation with one QSO remained visible after sign-out and re-login.

### Remaining Phase 6 evidence

Published award-layout editing, publishing and issuance after the current image update; credentialed Observe delivery; log/trace backends and workflow correlation; and the documented database/object-storage recovery limitations remain open under [myota-next #10](https://github.com/myota-platform/myota-next/issues/10) and [#91](https://github.com/myota-platform/myota-next/issues/91). Earlier participant profile/callsign and single-instance JetStream recovery evidence is recorded below.

### Observe provisioning procedure

The MyOTA Next chart currently defaults `observe.enabled` to `false`. To
qualify delivery, first create the `myota-observe` Secret in namespace
`myota-test` with the `app-key` and `app-secret` keys using the approved
secret store. Then set `observe.enabled: true` in the Fleet values and apply
the change through the protected pull-request and Rancher catalog workflow.
Do not commit the credential values. After rollout, verify stable service IDs,
instrumentation hooks, startup, and actual signal delivery in the configured
Observe destination. This procedure is pending because neither the Secret nor
approved credentials are currently available in the test namespace.

The three successful Actions runs establish build publication and the chart
upgrade. The new browser checks above verify the post-upgrade sign-in, public
catalogue, activation, and QSO path. They do not complete Phase 6 readiness or
authorize production cutover.

## Activity worker rollback qualification — 2026-10-27 02:34 UTC

The test release was deliberately rolled back to the previously deployed
Activity worker image, then restored through the protected Rancher catalog
workflow. This validates an image rollback while databases and shared services
remain at the current release.

- **Rollback:** PR [#141](https://github.com/myota-platform/myota-next/pull/141)
  merged as [`37f3b73`](https://github.com/myota-platform/myota-next/commit/37f3b73b435d783b96798bd872df8b270e517dd2).
  Protected deployment [40437074286](https://github.com/myota-platform/myota-next/actions/runs/40437074286)
  deployed chart `1.1.6-testing.37f3b73b435d.1` at Helm revision 36 and
  selected the prior worker image
  `sha256:07b206434addf75a4c1ee33cc47214e63e70c94cf679653c93abf5bf83c54597`.
- **Forward restore:** PR [#142](https://github.com/myota-platform/myota-next/pull/142)
  merged as [`b1bd3b9`](https://github.com/myota-platform/myota-next/commit/b1bd3b9bd7652037649bef1849333bab2616e5dd).
  Node CI passed on retry after one transient live callsign browser focus
  assertion; all 19 non-publishing image build/scan jobs passed. Protected
  deployment [40438930034](https://github.com/myota-platform/myota-next/actions/runs/40438930034)
  restored chart `1.1.6-testing.b1bd3b9bd765.1`, Helm revision 37, and worker
  image `sha256:eb1e4bc419b2ee459e40734f99ce0465e9a3a824bb2751cc8564516e5e871411`.
- **Post-restore health:** Fleet is Ready at `b1bd3b9`; all 16 Deployments
  are fully ready (21/21 replicas). Admin and Participant roots and gateway
  `/readyz` returned HTTPS 200. Authenticated Admin sign-in reached
  `/dashboard`; Participant sign-in and `/v1/identity/me` both returned 200.
- **Persisted state:** Before and after both worker changes, the Activity
  database had 2 failed and 11 succeeded jobs, no queued/running jobs, 1
  completed and 1 failed import, and 1 unresolved work dead letter. The
  `MYOTA_ACTIVITY_WORK` stream contained 0 messages/bytes; all six consumers
  had 0 pending and 0 ack-pending messages, with delivered and acknowledged
  floors equal. The unresolved dead letter was not replayed.
- **Scope and limitation:** There were no Activity database migration changes
  between the worker images. The rollback was performed only after proving
  that the stream and active database work were empty. This demonstrates an
  idle-state application-image rollback with the current schema; it does not
  qualify the old worker consuming queued/in-flight messages or a rollback
  across incompatible schema/object changes. The runbook rejects rollback
  unless work is quiescent and requires a forward-fix/settlement path otherwise.
  With this safe-abort procedure and the successful idle rollback verified,
  [issue #93](https://github.com/myota-platform/myota-next/issues/93) is closed.

This does not complete the remaining Phase 6 qualification work in
[issue #10](https://github.com/myota-platform/myota-next/issues/10).

### Rollback preflight and recovery

Use the protected pull-request and Rancher catalog workflow described above
to change image digests. Do not run an independent Helm upgrade on this
Fleet-managed release. Before authorizing a rollback, require no queued or
running Activity database jobs, no JetStream stream messages, and no pending
or unacknowledged consumer deliveries. Compare the same state after rollback.

Read-only verification commands used for this qualification:

```sh
kubectl -n myota-test get applications myota-test -o wide
kubectl -n myota-test get deploy myota-test-activity-worker -o jsonpath='{.spec.template.spec.containers[0].image}{"\t"}{.status.readyReplicas}/{.spec.replicas}{"\n"}'
curl --fail --silent --show-error https://myota-test.spainip.es/readyz

# In a separate terminal, stop the port-forward after querying:
kubectl -n myota-test port-forward pod/myota-test-nats-0 18222:8222
curl --fail --silent --show-error 'http://127.0.0.1:18222/jsz?streams=true&consumers=true&accounts=true' |
  jq '[.account_details[]?.stream_detail[]? |
       select(.name == "MYOTA_ACTIVITY_WORK") |
       {messages: .state.messages,
        consumers: [.consumer_detail[]? |
                    {name, num_pending, num_ack_pending}]}]'

# ACTIVITY_DATABASE_URL must be obtained through the approved secret store.
psql "$ACTIVITY_DATABASE_URL" -v ON_ERROR_STOP=1 <<'SQL'
SELECT status, count(*) FROM activity_job GROUP BY status ORDER BY status;
SELECT status, count(*) FROM activity_import GROUP BY status ORDER BY status;
SELECT count(*) AS unresolved
FROM activity_work_dead_letter WHERE resolved_at IS NULL;
SQL
```

If any database job is QUEUED or RUNNING, any stream message exists, or any
consumer has pending or unacknowledged messages, do not switch to the older
image. Keep the current digest selected and let compatible work settle. If the
current worker is unhealthy, ship a forward fix at the current schema and
reconcile database jobs through the owning Activity workflows. Keep completed
and failed job/import counts stable; do not replay an unresolved dead letter
as part of rollback. Repeat the preflight only after active work has been safely
settled. If safe settlement cannot be established, cancel rollback and keep
the known-good current release active.

## Prometheus scrape rollout — 2026-10-27 01:56 UTC

The optional Prometheus scrape configuration is now installed in the
Rancher-managed `myota-test` release. Two PRs corrected the test-only catalog
upgrader Role after the first upgrade attempts exposed its missing
ServiceMonitor permissions: [PR #139](https://github.com/myota-platform/myota-next/pull/139)
added `list`, and [PR #140](https://github.com/myota-platform/myota-next/pull/140)
added `watch`.
Both changes passed Node CI and image build/scan checks before merging to
`testing`. The Role remains restricted to the `myota-test` namespace and the
`monitoring.coreos.com/servicemonitors` resource.

- **Fleet source:** `fleet-local/myota-next-test` is Ready at commit
  [`fa7bbe5`](https://github.com/myota-platform/myota-next/commit/fa7bbe557ecfc3dda8abee35c6ac6eed39dcbcc3).
  `GitPolling=True`, `Reconciling=False`, and `Stalled=False`.
- **Rancher catalog:** The protected rollout
  [40435365127](https://github.com/myota-platform/myota-next/actions/runs/40435365127)
  succeeded. Helm release `myota-test` is deployed at revision 35 with chart
  `1.1.6-testing.fa7bbe557ecf.1`.
- **ServiceMonitors:** `myota-test-apis` scrapes the five APIs and
  `myota-test-workers` scrapes Activity notifications, Email, and three outbox
  workers. Both scrape `/metrics` every 30 seconds. Prometheus reports all 15
  discovered targets healthy (`up=1`), and the Prometheus API returned 321
  samples with the `myota-test` namespace labels.
- **Workload and endpoint health:** All 21 application replicas are ready.
  Admin and Participant web roots return HTTPS 200 with certificate
  verification. The unauthenticated `GET /v1/identity/me` API route returns
  403, as expected for this protected endpoint.
- **Observe distinction:** Prometheus scraping verifies metrics collection by
  the in-cluster Prometheus stack; it does not prove `@nestjs/observe` delivery.
  The `myota-observe` Secret is absent, so authenticated Observe delivery
  remains open under
  [issue #90](https://github.com/myota-platform/myota-next/issues/90).

This closes the ServiceMonitor/RBAC deployment issue but does not complete the
remaining Phase 6 qualification work in
[issue #10](https://github.com/myota-platform/myota-next/issues/10).

## Previous MyOTA Next rollout — 2026-10-26 23:58 UTC

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
- **Grafana through the Admin proxy:** An authenticated Admin browser session
  opened the embedded `/grafana` view. Grafana served the default dashboard
  with six panels; all 13 dashboard/search/query API responses and 49 CSS,
  JavaScript, and image assets returned HTTP 200, with no browser errors. The
  dashboard catalogue returned 52 dashboards and `/observability/api/health`
  returned JSON with database status `ok`. An unauthenticated request to the
  protected proxy returned HTTP 302. Grafana's own user and star APIs return
  401 because this shared instance uses anonymous dashboard access; those
  profile features are not part of the Admin dashboard qualification. No
  shared Grafana configuration was changed. See [issue #116](https://github.com/myota-platform/myota-next/issues/116).
- **Participant Activity journey:** A signed-in synthetic Participant used
  the browser UI to upload a one-record ADIF file against an open activation.
  Import `e5a1cc51-7542-47f2-989b-fdb55e9ff24b` completed with 1 record seen,
  1 accepted, and 0 rejected. Activity PostgreSQL contains one `VALID` QSO
  linked to that import, and the activation's QSO count is 1. This verifies a
  successful browser-to-worker-to-database import path.
- **Participant end-to-end Activity journey (2026-10-27 02:48 UTC):** On the
  protected deployment at chart revision 37 (`1.1.6-testing.b1bd3b9bd765.1`,
  Fleet commit `b1bd3b9`), the synthetic Participant signed in and started an
  activation on the approved synthetic QA entity. Activation creation and
  manual QSO entry returned HTTP 201. The browser submitted a one-record ADIF
  file (HTTP 202); the authenticated status query reached `COMPLETED` with 1
  accepted and 0 rejected records. The Participant closed the activation
  (HTTP 200), signed out, signed in again, and saw the persisted `CLOSED`
  activation with two QSOs. The synthetic activation and its two QSOs remain
  in `myota-test` as qualification data. This verifies the complete deployed
  browser-to-Gateway-to-Activity-worker path, but does not provision or approve
  the production judicial-district catalogue; programme rules and bulk entity
  assignment remain tracked separately in [issue #79](https://github.com/myota-platform/myota-next/issues/79).
- **Participant callsign self-service (2026-10-27):** The signed-in synthetic
  Participant added a secondary QA callsign through the profile UI (HTTP 201)
  and retired it through the same UI (HTTP 200). The profile reloaded and
  displayed the callsign as `RETIRED`; the existing primary callsign remained
  available. This left only a retired synthetic QA callsign record on the test
  account.
- **Participant manual QSO and activation closure:** On 2026-10-27, the same
  synthetic Participant account recorded a manual contact against the open
  activation (HTTP 201), closed it (HTTP 200), signed out, and signed back in.
  The browser and Activity response both showed `CLOSED` with two QSOs. This
  used synthetic test data; the activation is no longer open.
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
- **2026-10-27 runtime-role recheck:** an in-cluster read-only probe verified
  the current `myota-test` credentials against all three databases. Each
  owner-specific role (`myota_core_app`, `myota_activity_app`, and
  `myota_geo_app`) passed its own database checks; all six cross-owner
  attempts were denied. The probe also confirmed that runtime roles cannot
  read or modify the schema migration ledger. It inspected database catalogs
  only and read no application rows. The three application Deployment specs
  reference no migration/admin URL Secret; their URLs use only the matching
  owner roles. The release remained at chart revision 38 with 16/16
  Deployments ready. The temporary probe Job and ConfigMap were deleted.
- After that verified transition, the legacy shared `myota_app` login was
  retired on all three databases through the documented role-sync command.
  Catalog checks confirmed `NOLOGIN`, zero remaining sessions, no database
  `CONNECT`, no schema `CREATE`, and no application-table DML or migration-ledger
  read privilege. The owner-only runtime check was rerun successfully after
  retirement; all 16 Deployments remained ready and Gateway `/healthz` and
  `/readyz` returned HTTP 200. No application rows were read or changed. The
  temporary Jobs and ConfigMaps were removed. The first combined verification
  attempt used migration URLs for the runtime probe and failed its role-name
  assertion; it made no data changes. The separate corrected runtime probe
  passed.
- [Issue #89](https://github.com/myota-platform/myota-next/issues/89): database
  and object-store snapshots were restored and integrity-checked. The
  repository PostgreSQL backup/restore wrapper also passed against newly
  created disposable databases on each isolated owner instance using
  PostgreSQL 16 tools: it created checksummed custom-format dumps, rejected an
  incorrect restore confirmation, restored mutated marker values, and removed
  its probe tables, backup directory, and scratch databases. On 2026-10-27,
  all 16 application Deployments were quiesced for a coordinated three-database
  and six-bucket snapshot, then restored into a separate PostGIS target and six
  empty mapped buckets. Exact row counts matched in all 139 restored tables;
  five objects (45,173,156 bytes) passed SHA-256/size verification, and both
  persisted ADIF object references resolved through the bucket mapping. The
  deployment returned to 21 available replicas and its public health endpoints
  returned HTTP 200. The full procedure and remaining limits are recorded in
  the [backup/restore evidence](database-object-backup-restore.md#2026-10-27-quiesced-database-and-object-restore).
  Off-node retention, an accepted schedule/RPO, realistic RTO, and PVC-loss
  recovery remain unqualified; the current migration URL role is privileged,
  so a least-privilege backup identity is still needed.
- [Issue #90](https://github.com/myota-platform/myota-next/issues/90):
  `@nestjs/observe` dependencies, root hooks, stable service IDs, and the
  no-credentials startup path are verified. The authenticated Grafana proxy
  and dashboard path are also live-qualified under [issue #116](https://github.com/myota-platform/myota-next/issues/116);
  this does not prove Observe telemetry ingestion. Authenticated Observe
  credentials and delivery to an Observe destination are not configured or
  verified.
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
  valid QSO. On 2026-10-27, the browser also recorded a manual QSO and closed
  that activation; the `CLOSED` state and two-QSO total persisted across
  sign-out and sign-in. Participant registration throttle behavior was
  exercised against the live endpoint. PR #136 merged to `testing` as
  [`1083718`](https://github.com/myota-platform/myota-next/commit/1083718a362c1bec66bb9386d95c245ae53b0b0d);
  post-merge Node CI [40427700456](https://github.com/myota-platform/myota-next/actions/runs/40427700456)
  and the changed-image publisher [40427700526](https://github.com/myota-platform/myota-next/actions/runs/40427700526)
  succeeded. The PR adds Identity safeguards and invalid-link browser
  regressions. Correlated
  telemetry/audit evidence and QSO correction/edit behavior remain
  unqualified. See the [Participant qualification record](evidence/phase6-participant-registration.md).
- [Issue #93](https://github.com/myota-platform/myota-next/issues/93): a
  previous Activity worker digest was deployed and then restored through the
  protected Rancher catalog path. Admin and Participant sign-in, readiness,
  database aggregates, and empty JetStream consumer state were checked before
  and after. No Activity migration changes separated the images. This only
  qualifies rollback while work is quiescent; it does not test the older
  worker consuming pending/in-flight work. The runbook requires aborting
  rollback when active work remains and following a forward-fix/settlement path.
  Issue #93 was closed against that explicit safe-abort criterion; schema/object
  incompatibility beyond the tested same-schema worker rollback remains outside
  the proven procedure.

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
