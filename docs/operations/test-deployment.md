# Isolated test deployment

**Verified:** 2026-10-13 19:56 UTC
**Cluster:** local single-node K3s on `spainip-k3s`
**Namespace:** `myota-test`
**Fleet source:** `myota-platform/myota-deploy`, branch `testing`, commit
[`5d091919`](https://github.com/myota-platform/myota-deploy/commit/5d091919b9d9026146777b001f6af8e80527bf93)

This is a separate Helm release with its own PostgreSQL, JetStream, SeaweedFS,
and observability storage. It is not a production replica service: the initial
copy is a one-time seed, and production data is not continuously replicated.
The live `myota` namespace and its persistent volumes were left unchanged.

## Deployment status

| Check | Result |
|---|---|
| Helm | Release `myota-test`, chart `myota-test-0.1.0`, revision 10, `deployed`. |
| Fleet | GitRepo Ready; commit `5d091919`; 75/75 resources Ready. |
| Workloads | Every test Deployment and PostgreSQL StatefulSet has all desired replicas ready. |
| Admin UI | [admin.test.myota.top](https://admin.test.myota.top) returns HTTP 200 with a publicly trusted ACME certificate. |
| Participant UI | [test.myota.top](https://test.myota.top) returns HTTP 200 with a publicly trusted ACME certificate. |
| API ingress | `api.test.myota.top` terminates with a valid ACME certificate; `/` returns 404 because the gateway does not serve a root route. |

Fleet watches the Deploy `testing` branch and corrects drift. The test values
use the published `testing` image tags and immutable image digests, so a new
Helm digest value causes Kubernetes to roll the corresponding workload.

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
require a deliberate digest update on `testing`.

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
key and size: 10 matching objects totaling 13,625,617 bytes.

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
