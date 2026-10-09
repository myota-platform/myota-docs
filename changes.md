# MyOTA changes

Newest deliveries first. Earlier reconstructed service-by-service milestones
remain in the [implementation timeline](docs/history/implementation-timeline.md).

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

Deployment and final CI results are recorded in the evidence page after rollout.
