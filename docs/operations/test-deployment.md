# Isolated test deployment

**Current verification:** 2026-10-14 20:02 UTC
**Initial seed verification:** 2026-10-14 11:09 UTC
**Cluster:** local single-node K3s on `spainip-k3s`
**Namespace:** `myota-test`
**Current Fleet source:** `myota-platform/myota`, branch `testing`, commit
[`5b33466`](https://github.com/myota-platform/myota/commit/5b33466155bf0bd3bd69a65fbfdc6d37a338b812), path `deploy/helm/myota-test`.

This is a separate Helm release with its own PostgreSQL, JetStream, SeaweedFS,
and observability storage. It is not a production replica service: the initial
copy is a one-time seed, and production data is not continuously replicated.
The live `myota` namespace and its persistent volumes were left unchanged.

## Current monorepo rollout — 2026-10-14 20:02 UTC

| Check | Result |
|---|---|
| Fleet | The Testing-only GitRepo `fleet-local/myota-deploy-testing` reads `myota-platform/myota`, branch `testing`, path `deploy/helm/myota-test`, at commit [`5b334661`](https://github.com/myota-platform/myota/commit/5b33466155bf0bd3bd69a65fbfdc6d37a338b812). Its bundle deployment is Ready. Production Fleet remains on `myota-platform/myota-deploy`, branch `main`, at `40a4bdb6be4e75146f0d74ab440db41e98c5d372`. |
| Helm and storage | Release `myota-test` is deployed at revision 304. All 10 database and observability PVCs in `myota-test` are Bound. |
| Workloads | All 24 Deployments and all 3 StatefulSets report their desired replicas ready. |
| MyOTA service health dashboard | Grafana's `myota-observability` ConfigMap and mounted dashboard file contain `myota-service-health.json` with title **MyOTA service health** and UID `myota-service-health`. It is provisioned in the **MyOTA** folder and uses the `MyOTA Prometheus` datasource (`myota-prometheus`). Live Prometheus queries return `myota_identity_users` values `OPERATOR=2` and `SWL=0`, and `myota_programmes=2`; these match the corrected dashboard expressions. The old `myota-operations` dashboard was retired and is not the current dashboard. |
| Logs dashboard | The deployed `myota-observability` ConfigMap contains `myota-logs-failures.json` with UID `myota-logs-failures`; Grafana logged `finished to provision dashboards` at 19:42 UTC. It uses the MyOTA Loki datasource. |
| Trace dashboard | The deployed `myota-observability` ConfigMap contains `myota-traces.json` with UID `myota-traces`. Grafana's read-only dashboard search returned `MyOTA Distributed Traces`; the Tempo datasource `myota-tempo` is configured. |
| Existing telemetry | A read-only Tempo search returned 20 stored traces across Activity, Geodata, and outbox services. The current failed-trace and over-one-second queries each returned zero; no synthetic traces were inserted. |
| Digest workflow | The workflow code is `.github/workflows/sync-testing-image-digests.yml` in `myota`. It executes on GitHub-hosted `ubuntu-latest` runners; the Kubernetes dispatcher is only a planned trigger and no digest-dispatch CronJob is installed in `myota-test`. Manual run [#38625702955](https://github.com/myota-platform/myota/actions/runs/38625702955) completed as `workflow_dispatch` on `main`, but skipped because the repository variable `TESTING_DIGEST_APP_ID` and secret `TESTING_DIGEST_APP_PRIVATE_KEY` are unset. It created no digest PR. |
| Recent Admin image | Testing push [#38625361907](https://github.com/myota-platform/myota/actions/runs/38625361907) passed and published the Admin image. Its updated digest has not been recorded or reconciled because the digest-writer App is not configured. |

The Logs / Failures and Distributed Traces dashboards are provisioned in
Grafana's **MyOTA** folder. Both default to a 30-minute time window and a
30-second refresh. The 19:49 UTC read-only recheck found the log dashboard
ConfigMap payload SHA-256
`c664a46c452a042ee858966f7c3f163bff8d18ec971ea9577f2b768a7b98f062` and
trace dashboard payload SHA-256
`0a97abe5410b4a58c7871ffcaaca4d3e22314c842ea74ab586090bb7587a50be`. The
configured Loki and Tempo datasource UIDs were `myota-loki` and `myota-tempo`.
The last recorded bounded Tempo search found 20 traces, but none matched the
failed-trace or over-one-second filters; no synthetic traces were inserted.

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
