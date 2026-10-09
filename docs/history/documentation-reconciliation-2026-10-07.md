# Organization documentation reconciliation — 7 October 2026

Scope: reconcile the latest published geodata scaling, admin-client and
JetStream operations delivery, then check every organization repository.
Before this update, all twelve local `main` branches were clean and matched
their fetched `origin/main`; the gap was documentation consistency, not
unpublished runtime implementation.

## Repository coverage

| Repository | Reconciled documentation |
|---|---|
| `.github` | Public scaling checklist, completed row-authority/replay work versus outstanding failure/memory gates |
| `myota-docs` | [Detailed roadmap and evidence](../geodata/horizontal-scaling-roadmap.md), [ownership map](../architecture/repository-map.md), [operations](../operations/runbooks.md), [observability](../observability/overview.md), import and service-boundary diagrams |
| `myota-geodata-service` | Three durable domain consumers, archived snapshot semantics, verified concurrency and remaining streaming/restart gates |
| `myota-admin-web` | Candidate sources, separate import workspace, revision-aware edits, operations-page boundaries and evidence links |
| `myota-operations-service` | Authenticated broker inspection, sampled history and domain-worker ownership links |
| `myota-contracts` | Correct standalone validation commands, both runtime registries and current API/client evidence |
| `myota-deploy` | Five service ports, three geodata consumers, failure-versus-freshness alert semantics and authoritative architecture links |
| `myota-platform` | Current integration role, three-database architecture, lifecycle diagram, deployment pointers and migration mirrors |
| `myota-identity-service` | Actual standalone entry point, core database and operations permissions |
| `myota-programme-service` | Actual standalone entry point, core storage and geodata/programme ownership |
| `myota-activity-service` | Actual standalone entry point, activity-only data ownership and deletion-cascade API boundary |
| `myota-web` | Static-client deployment instructions, API-only access and removal of built-in entity seed claims |

## Validation and mirror repair

- [x] Reconcile OpenAPI and runtime routes: 162 contract operations, 259 route
  registrations, no missing routes or duplicate operation IDs; the seven
  existing semantic-duplicate candidates remain explicitly tracked.
- [x] Validate both checked-in clients and their 18 preferred operations.
- [x] Compare all 42 geo, activity and operations deployment/bootstrap migration
  mirrors byte for byte against their service-owned sources.
- [x] Restore missing platform activity mirrors `003_adif_source_retention.sql`
  and `004_adif_failed_source_retention.sql`. The platform runner already
  referenced them and deploy already carried them. No new SQL behavior,
  retention policy or live database migration is introduced.
- [x] Run the 20 platform compatibility regressions and migration-runner shell
  syntax check successfully.
- [x] Check local documentation targets and GitHub file/heading links against
  the corresponding checkouts; check changed files for whitespace errors.

Runtime/Colima and CI evidence from the implementation delivery is linked in
the [roadmap delivery table](../geodata/horizontal-scaling-roadmap.md#latest-delivery-and-evidence--8-october-2026).
This documentation reconciliation does not rerun production load tests,
increase replicas, redeploy the stack or close forced-failure/streaming/canary
gates. Migration ownership and the synchronization rule remain in the
[repository map](../architecture/repository-map.md#documentation-synchronization-rule).
