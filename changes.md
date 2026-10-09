# MyOTA changes

Newest deliveries first. Earlier reconstructed service-by-service milestones
remain in the [implementation timeline](docs/history/implementation-timeline.md).

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
  jobs.
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
