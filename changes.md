# MyOTA changes

Newest deliveries first. Earlier reconstructed service-by-service milestones
remain in the [implementation timeline](docs/history/implementation-timeline.md).

## 9 October 2026 — geodata Phase 3 bounded preprocessing (partial)

- **Geodata service:** Added streamed object-to-scratch downloads and an
  incremental `ijson` path for uploaded GeoJSON FeatureCollections/arrays.
  Candidate persistence now checkpoints by stable ordinal in configurable
  100-feature windows and evicts committed row projections before advancing.
  Compose and Helm expose the batch-size setting. Replay/cancellation and
  source metadata behavior remain covered by unit regressions.
- **Verification boundary:** Ruff passed; the local suite had 104 passes and
  17 environment-dependent skips (121 collected); the focused import/worker suite passed
  32 tests with its `ijson`-dependent test skipped locally. This host could not
  reach PyPI, so local verification used the parser compatibility fallback.
  The actual `ijson` path passed in the [PostGIS-backed GitHub quality run](https://github.com/myota-platform/myota-geodata-service/actions/runs/37913194172),
  which installed service requirements and passed all 121 tests without skips.
  Whole-document fallbacks remain for KML, GPX, Shapefile and snapshot imports.
  No peak-RSS measurement, worker termination/forced reclaim, or concurrent
  worker proof exists yet; Phase 3 remains open.
- **Evidence:** [Phase 3 implementation and exit review](docs/geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
  [horizontal-scaling roadmap](docs/geodata/horizontal-scaling-roadmap.md),
  [geodata service commit 301bc28](https://github.com/myota-platform/myota-geodata-service/commit/301bc28),
  and [deployment commit 890197f](https://github.com/myota-platform/myota-deploy/commit/890197f).
  Service validation passed locally: 104 passed and 17 environment-dependent
  tests skipped; Ruff passed. The streaming-dependency test was skipped locally
  because this host could not reach PyPI. Updated root status, geodata index,
  WIP, To do, and prioritized backlog with the precise partial status.
  [image publication](https://github.com/myota-platform/myota-geodata-service/actions/runs/37913192906)
  and [Helm chart validation](https://github.com/myota-platform/myota-deploy/actions/runs/37913227459)
  passed. Fleet observed deployment commit `890197fa4f00b828a0be3b1b4ab4b645471ba89f`;
  BundleDeployment reached `Ready=True`, Helm revision 118 is deployed, and
  the API and worker are each 1/1 ready. The worker uses batch size 100 and
  the published service image digest recorded in the linked evidence report.
  The gateway health check passed. No production import or worker fault
  injection was performed; Phase 3 remains open pending bounded-path and
  worker-recovery evidence.

## 9 October 2026 — geodata Phase 2 upload recovery gate

- **Geodata service:** Added a repeatable CI failure-injection scenario pinned
  to the exact SeaweedFS image ID observed in the live K3s pod. In an isolated
  test environment, it restarts the API between multipart parts and restarts
  the SeaweedFS container on a disposable persistent volume; verifies resume,
  per-part and whole-object checksums, import metadata/outbox durability,
  idempotent retry, owner isolation and abort cleanup. Production storage was
  not restarted or written. The service README now links the runbook/evidence.
- **Evidence:** [Phase 2 recovery report](docs/geodata/evidence/phase2-upload-recovery-2026-10-09.md);
  [geodata service commit 853fcbc](https://github.com/myota-platform/myota-geodata-service/commit/853fcbc5b30085c8a450ff0accbe32e414348c22);
  [GitHub Actions run 37908154059](https://github.com/myota-platform/myota-geodata-service/actions/runs/37908154059)
  passed the recovery, relational-boundaries, load-harness and image-publishing
  jobs. The follow-up service documentation/image build also passed in
  [run 37908394130](https://github.com/myota-platform/myota-geodata-service/actions/runs/37908394130).
- **Helm rollout:** Digest sync was committed as
  [`myota-deploy` 159861e](https://github.com/myota-platform/myota-deploy/commit/159861e1d77461882f49b7ff44859ce61ebd1845).
  [GitHub Helm render](https://github.com/myota-platform/myota-deploy/actions/runs/37908782124)
  passed. Fleet reconciled the commit with `Ready=True`; Helm revision 115 is
  deployed, the new geodata image is Ready, all 21 deployments are Ready, and
  the public gateway health check succeeded. Detailed image and deployment IDs
  are in the [Phase 2 evidence report](docs/geodata/evidence/phase2-upload-recovery-2026-10-09.md).
- **Roadmap:** Phase 2 is complete for the tested digest. Phase 3 bounded
  parsing/worker-failure proof, Phase 4 infrastructure review, and Phase 5
  broader rollout proof remain open. Updated the geodata index, evidence index,
  work-in-progress, to-do/prioritized backlog and retained Phase 2 prompt to
  distinguish completed recovery from future failure scenarios.

## 9 October 2026 — administration workspace reorganization

- **Admin web:** Permission-aware searchable navigation, guarded direct links,
  consistent headings/refresh/actions, explicit programme scope and task entry
  points. Entity categories join Entities; policies and certificate design are
  distinct. Users, roles and security have separate tabs.
- **Admin forms:** Preserve scoped user-role assignments and clear password
  reset fields after saving; immutable roles/published versions and unprivileged
  award/category edits are read-only. Report failures separately from success;
  unavailable metrics are not fabricated zeros. Map loading is explicitly paged.
- **Compatibility:** Existing API resources, routes, map/import/deletion/award
  workflows, UTC and service data boundaries remain. No schema change.
- **Documentation:** [Workspace guide](docs/domain/administration/navigation-reorganization.md),
  [diagram](docs/architecture/diagrams/admin-workspaces.md) and
  [validation/deployment evidence](docs/domain/administration/evidence/admin-workspaces-2026-10-09.md).

Final delivery: 30 unit tests, 23 browser checks, production build/types and
GitHub Helm rendering passed. Live read-only verification covered 14 pages with
zero browser/API failures or attempted writes. Helm **0.2.12 revision 114** is
deployed, Fleet **1/1 Ready**, and all 21 MyOTA deployments are Ready. See the
evidence page for commits, workflow links, image digest and verification limits.
