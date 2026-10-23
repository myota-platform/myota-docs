# Current MyOTA repository and source ownership map

## Migration status

[`myota-next`](https://github.com/myota-platform/myota-next) is now the
canonical repository for ongoing implementation work on the clean Node.js and
NestJS platform. Its function-parity baseline and current status are recorded
in the repository's [status page](https://github.com/myota-platform/myota-next/blob/feature/nestjs-runtime-foundation/docs/status.md).
The original [`myota`](https://github.com/myota-platform/myota) monorepo and
the former service repositories remain migration inputs and behavioral
references; new implementation work for the rebuild belongs in `myota-next`.
This source-of-code change does not move live deployment ownership:
Production and Testing Fleet still use `myota-deploy`. Former repositories
remain available until deployment cutover is separately verified. The
[transition evidence](evidence/monorepo-transition-2026-10-14.md) records exact
commits, runs, issue migration, live Fleet paths, and exit criteria.

## Target source layout

| Domain or responsibility | Target path in `myota-next` | Ownership boundary |
|---|---|---|
| Identity | `apps/identity` | Accounts, authentication, roles, scopes, and OIDC mappings; owns Core database objects in its scope. |
| Programme | `apps/programmes` | Programme data, shared entity categories, assignments, policy, and themes; owns its Core schema. |
| Geodata | `apps/geodata` | PostGIS entities, imports, provenance, review, audit, and durable workers; owns its separate Geodata database schema. |
| Activity | `apps/activity` | Activations, QSOs, awards, corrections, notifications, and workers; owns its separate Activity database schema. |
| Participant UI | `web/participant` | Participant-facing web client; no domain database ownership. |
| Admin UI | `web/admin` | Administration workflows; coordinates APIs and owns no domain data. |
| Contracts | `contracts` | OpenAPI, event registry/schemas, compatibility rules, and generated client. |
| Runtime/deployment | `deploy` | Compose, Helm, migration orchestration, image builds, and environment configuration. |
| Cross-service checks | `tools` | Cross-service integration, compatibility, and smoke checks; not a service data owner. |
| Email delivery | `apps/email-worker` | Core-owned durable email delivery worker; shares Core ownership with Identity and Programmes. |

The repository consolidates source management, not runtime processes. Keep
services separately deployable and scalable. Preserve independent database
ownership, API behavior, authorization, data retention, failure handling, and
image/deployment boundaries. Cross-service references remain IDs and
versioned APIs/events; do not introduce cross-database foreign keys.

## Repository disposition during transition

| Repository | Current disposition | Cutover condition |
|---|---|---|
| `myota-next` | Canonical source for ongoing clean-platform implementation; active work is on feature branches. It is not yet the live deployment source. | Complete any remaining functional slices, then perform a separately approved deployment transition through the isolated test environment before production release. |
| `myota` | Existing monorepo retained as a behavioral reference and migration input; do not land new rebuild implementation work here. | Preserve the repository and its history until its runtime and deployment responsibilities have been independently dispositioned. |
| `myota-identity-service`, `myota-programme-service`, `myota-geodata-service`, `myota-activity-service` | Imported source snapshots; existing production image packages and live systems still depend on prior deployment sources. | Prove code/migration parity, publish from monorepo, deploy through Testing and production, then archive. |
| `myota-web`, `myota-admin-web` | Frontend source snapshots are in `web/participant` and `web/admin`. | Verify builds, package access, deployed frontend routing, and release ownership before archival. |
| `myota-deploy` | Current production and Testing Fleet source; its Compose/Helm assets were copied to `deploy/`. | Move Fleet and digest automation to the monorepo, validate both environments, then archive. |
| `myota-platform` | Existing integration/bootstrap and synchronized Deploy/runtime mirrors. | Verify required integration coverage and mirror disposition. Remove redundant copies only after comparison. |
| `myota-contracts` | Contract source was copied to `contracts/`; monorepo contract CI passes. | Verify schema/client/release parity and consumer references, then archive after all users consume monorepo contracts. |
| `myota-docs` | Separate documentation, ADR, operations, and evidence owner; explicitly retained. | Remains separate. |
| `.github` | Separate organization governance, contribution policy, profile, and shared digest automation owner; explicitly retained. | Remains separate; keep links pointed to the monorepo and canonical Docs. |
| `myota-operations-service` | Archived and retired on 2026-10-13. | No further runtime migration. |

The issue counts below describe the historical issue migration into the
original `myota` repository as of 2026-10-14. They do not describe or transfer
the issue backlog for `myota-next`.

Issue migration was a tracking move: the 49 formerly open issues were copied
with their acceptance criteria and reciprocal links, and their source issues
were closed as migrated. They are not completed work. The original monorepo had
50 open issues at that snapshot, including the GHCR permission gate.

## Deployment-source and mirror rule

The source of deployed configuration has not moved yet. Fleet currently tracks
`myota-deploy/main` at `deploy/helm/myota` and
`myota-deploy/testing` at `deploy/helm/myota-test`. `myota-next/deploy` is the
candidate deployment source for the clean implementation, but it must pass the
authorized image-publication, isolated Helm deployment, and rollout checks
before any Fleet source changes. Do not archive Deploy or remove Platform
copies until then.

During transition:

1. Keep the current live deployment reproducible from the existing release
   source.
2. Treat `myota-next/deploy` as the migration candidate until Testing and
   production cutovers are proven.
3. Compare each Platform mirror against its owner before removing or changing
   it; do not independently evolve duplicate Helm, migration, or runtime
   configuration.
4. Keep production and Testing digests separate. Testing tags and values must
   never enter production.
5. Record exact source commits, image digests, Helm/Fleet status, checks, and
   cleanup evidence before marking a gate complete.

## Documentation synchronization

`myota-docs` remains authoritative for cross-service architecture, decisions,
operational procedures, and evidence. The `.github` profile is the concise
organization summary and issue index. The
[monorepo transition evidence](evidence/monorepo-transition-2026-10-14.md) is
the detailed record for this cutover; service and domain plans remain
authoritative for their own behavior and acceptance criteria.

When a source boundary, contract, deployment, schema, or user capability
changes, update the relevant implementation documentation, this map, the
current-state record, affected diagrams, and the `.github` profile links.
