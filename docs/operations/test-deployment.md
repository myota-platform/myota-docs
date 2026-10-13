# Isolated test deployment

**Verified:** 2026-10-13 20:34 UTC
**Cluster:** local single-node K3s on `spainip-k3s`
**Namespace:** `myota-test`
**Fleet source:** `myota-platform/myota-deploy`, branch `testing`, commit
[`d0556c3`](https://github.com/myota-platform/myota-deploy/commit/d0556c376dbaa830d4bc2508a04252dfa496a777).

This is a separate Helm release with its own PostgreSQL, JetStream, SeaweedFS,
and observability storage. It is not a production replica service: the initial
copy is a one-time seed, and production data is not continuously replicated.
The live `myota` namespace and its persistent volumes were left unchanged.

## Deployment status

| Check | Result |
|---|---|
| Helm | Release `myota-test`, chart `myota-test-0.1.0` with shared chart dependency `0.2.25`, revision 4, `deployed`. |
| Fleet | GitRepo and BundleDeployment Ready at Deploy `testing` commit `d0556c376dbaa830d4bc2508a04252dfa496a777`. |
| Workloads | Every test Deployment and PostgreSQL StatefulSet has all desired replicas ready. |
| Images | All ready MyOTA application containers use immutable testing-tag digests, and each running container image ID matches its declared digest. |
| JetStream | The deployment-owned provisioner validated all three streams and ten selected work durables. |
| Admin UI | [admin.test.myota.top](https://admin.test.myota.top) returns HTTP 200 with a valid ACME certificate. |
| Participant UI | [test.myota.top](https://test.myota.top) returns HTTP 200 with a valid ACME certificate. |
| API ingress | `api.test.myota.top` terminates with a valid ACME certificate; `/` returns 404 because the gateway does not serve a root route. |

Fleet watches the Deploy `testing` branch and corrects drift. The test values
use the published `testing` image tags and immutable image digests, so a new
Helm digest value causes Kubernetes to roll the corresponding workload.
Deploy PR [#44](https://github.com/myota-platform/myota-deploy/pull/44) updated
the platform and gateway digests to the images published by testing commit
`d03c3d39`. Fleet reconciled the merged values; all 27 Deployments and
StatefulSets are ready, all running application image IDs match their configured
digests, and both UI endpoints still return HTTP 200 with valid TLS.

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

The digest polling and protected-branch PR workflow is present on Deploy's
`testing` branch. GitHub scheduled workflows run from the repository's default
branch (`main`), where the existing scheduled workflow updates production
image values. A manual `workflow_dispatch` run on `testing` (run
[38457930823](https://github.com/myota-platform/myota-deploy/actions/runs/38457930823))
verified the registry lookup but failed when creating its protected-branch PR:
the organization policy prohibits GitHub Actions from creating or approving
pull requests. Repository-level permission changes are rejected by the
organization, and this session has no organization policy administration
scope. Therefore the testing digest poller is **not active yet**. A testing
image publication will not independently update this release until a safe
release pull request places the test poller on `main` and an organization
administrator permits the workflow to create pull requests. The Deploy `main`
and `testing` branches are divergent; promoting the whole testing branch would
include unrelated production configuration and migration changes, so it was
not done as part of this isolated deployment. Until these gates are resolved,
Fleet automatically reconciles committed test values but new image digests
require a deliberate digest update on `testing`. The current deployment has
the latest digests published by the testing builds checked by PR #44; this
does not enable automatic updates for future image publications.

## Initial data seed

The database and object data were copied after migrations and before enabling
public test ingress. Identity accounts were zero in the source, so the
relational seed's account guard passed without copying users or credentials.
Verified source/target row counts were:

| Data | Source and test result |
|---|---:|
| Core `public.account` | 0 rows |
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

- The authoritative Helm source and Fleet bundle are in
  [Deploy `deploy/helm/myota-test`](https://github.com/myota-platform/myota-deploy/tree/testing/deploy/helm/myota-test).
- The test release uses `values-testing.yaml` and
  `values-image-digests-testing.yaml`; production values remain under
  `deploy/helm/myota`.
- Re-run the one-time seed only after reviewing the per-database markers and
  target rows. The object copy is safe to repeat because it overwrites matching
  keys, but it does not delete target-only objects.
- This snapshot proves readiness and endpoint responses at the stated time.
  It does not qualify single-node failure recovery, production capacity, or
  continuous data synchronization.
