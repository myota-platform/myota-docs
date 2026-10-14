# Monorepo transition evidence — 14 October 2026

**Status:** Integration groundwork is in `myota/testing`; release and source-repository cutover are incomplete.
**Checked:** 2026-10-14 16:02 UTC
**Scope:** Current-main source snapshots, monorepo layout, CI/Helm workflows, issue migration, GitHub publication permissions, and live Fleet source references. No runtime behavior, API contract, database schema, or cluster resource was changed.

## Decision and boundaries

The monorepo follows the accepted
[assessment](../monorepo-and-scalability-assessment.md): one application source
repository with distinct services, APIs, databases, images, workers, and
deployment boundaries. Python and TypeScript remain unchanged. Docs and
organization governance stay in their own repositories. The Contracts source
repository remains during transition while the copied contracts are qualified.

The repository was initialized from the current `main` snapshots. No prior
branches or source commit histories were migrated. The root layout is:

| Area | Monorepo path |
|---|---|
| Identity | `apps/identity` |
| Programme | `apps/programmes` |
| Geodata | `apps/geodata` |
| Activity | `apps/activity` |
| Email placeholder | `apps/email` |
| Participant web | `web/participant` |
| Admin web | `web/admin` |
| Contracts and generated client | `contracts/` |
| Deployment, Compose, Helm, runtime images | `deploy/` |
| Cross-service tests | `integration/` |

Each service continues to own its database and migrations. Compose, Helm,
runtime environments, OCI package names, and data ownership stay separately
scoped. Platform Deploy copies and other duplicate runtime sources remain
transition artifacts until a checked parity decision removes them.

## GitHub changes

- User-created target repository:
  [`myota`](https://github.com/myota-platform/myota).
- Initial snapshot import commit:
  [`5e68724`](https://github.com/myota-platform/myota/commit/5e68724081494bbb7749f909675147af79f9589c).
- Layout normalization on `main`:
  [`47267d2`](https://github.com/myota-platform/myota/commit/47267d27ee66f08d6ed72220b0c27ec0c0ca6490).
- Feature PR [#3](https://github.com/myota-platform/myota/pull/3) merged to
  `testing` as
  [`a932f2f`](https://github.com/myota-platform/myota/commit/a932f2f3c511dbaebb27315e4c55646dbaec737b).
- PR #3's ten component, contracts, deployment, and Helm validation checks
  passed at head
  [`47ce0bb`](https://github.com/myota-platform/myota/commit/47ce0bb9722c0d858e01dbf0a4c9f42e34a3d100).
- The post-merge `testing` workflows reran the checks. Identity, Programme,
  Activity, Geodata, Admin, Participant, and deployment runtime tests passed.
  The post-merge image publication step failed for each of eight existing
  GHCR packages with `denied: permission_denied: write_package`.
  Contract checks passed. The chart and deployment checks passed on the PR
  head; the post-merge deployment runtime test also passed.

| Workflow | Result | Evidence |
|---|---|---|
| Identity | Tests pass; image push denied | [run 38594012176](https://github.com/myota-platform/myota/actions/runs/38594012176) |
| Activity | Tests pass; image push denied | [run 38594012258](https://github.com/myota-platform/myota/actions/runs/38594012258) |
| Participant web | Tests/build pass; image push denied | [run 38594012275](https://github.com/myota-platform/myota/actions/runs/38594012275) |
| Deployment runtime and images | Runtime tests pass; gateway/shared-runtime pushes denied | [run 38594012452](https://github.com/myota-platform/myota/actions/runs/38594012452) |
| Admin web | Test/build passes; image push denied | [run 38594012417](https://github.com/myota-platform/myota/actions/runs/38594012417) |
| Geodata | Qualification suites pass; image push denied | [run 38594012637](https://github.com/myota-platform/myota/actions/runs/38594012637) |
| Contracts | Pass | [run 38594012569](https://github.com/myota-platform/myota/actions/runs/38594012569) |
| Programme | Tests pass; image push denied | [run 38594012849](https://github.com/myota-platform/myota/actions/runs/38594012849) |
| Helm validation | Pass on PR head | [run 38592308739](https://github.com/myota-platform/myota/actions/runs/38592308739) |

The blocked image packages are `myota-identity-service`,
`myota-programme-service`, `myota-geodata-service`,
`myota-activity-service`, `myota-gateway`, `myota-service`,
`myota-admin-web`, and `myota-web`. Workflows already request
`packages: write`; package-level Actions access must also grant the new
repository write permission or the owner must approve a consistent package
rename/visibility strategy.

This blocker is tracked in
[myota issue #53](https://github.com/myota-platform/myota/issues/53). Do not
interpret the pre-existing mutable `testing` tags as newly published by the
monorepo. The new build did not update them.

The `.github` testing digest workflow now checks out `myota` and proposes a
digest-only PR against its `testing` branch. Its writer App installation and
credentials on the new repository have not been verified. The separate
workflow-dispatch App, if enabled through the planned CronJob, remains scoped
to `.github`; it is not the digest-writer App. See
[myota #7](https://github.com/myota-platform/myota/issues/7) and
[myota #8](https://github.com/myota-platform/myota/issues/8).

## Open-issue migration

The 49 open issues found across the source repositories were copied to
`myota` with their user stories, acceptance criteria, and related links.
Each source issue was closed as migrated and has a reciprocal link to its
monorepo issue. A fresh GitHub issue search found zero open issues in the
former source repositories, and 50 open issues in `myota` after adding the
GHCR permission blocker. The grouped live index is in
[.github/profile/README.md](https://github.com/myota-platform/.github/blob/main/profile/README.md).

Closing a source issue as migrated does not mean its task is complete. The new
monorepo issue remains the work record until its original acceptance criteria
are verified.

## Live release and repository state

A read-only local K3s inspection at 16:02 UTC found Fleet still tracking:

- production: `myota-deploy`, branch `main`, path
  `deploy/helm/myota`;
- Testing: `myota-deploy`, branch `testing`, path
  `deploy/helm/myota-test`.

No Kubernetes objects, Helm releases, databases, or application data were
changed for this migration. Existing production and Testing therefore continue
to use the split-repository release sources and previously deployed digests.
No monorepo release PR to `main` was opened because the artifact publication
gate is failing.

The connector denied branch-protection reads for `myota/main` and
`myota/testing` with HTTP 403 (`Resource not accessible by integration`).
It also exposes no repository-archive write action. The new branch protections
are therefore unverified, and source repositories have not been archived.
Do not archive a source repository while Fleet or the release process depends
on it.

## Exit criteria

The monorepo migration is not complete until all of the following are verified:

- [ ] Grant the monorepo's Actions identity write access to the eight existing
      GHCR packages, or adopt and test an approved package-name/visibility
      migration.
- [ ] Pass the full `testing` workflow set and publish both mutable
      `testing` and unique commit-SHA tags for all deployable components.
- [ ] Configure and verify repository rulesets: feature/fix PRs to
      `testing`, required quality gates, and release PRs to `main`.
- [ ] Update the digest-writer App installation/permissions and verify a
      digest-only PR to `myota/testing`.
- [ ] Point the isolated Testing Fleet source at `myota/testing` and
      `deploy/helm/myota-test`; verify Helm/Fleet readiness and both UIs
      without copying or modifying test data.
- [ ] Run production release validation through a PR from monorepo
      `testing` to `main`, update production Fleet only after all checks and
      rollout safeguards pass, and verify current workloads.
- [ ] Verify full Contracts parity and the remaining Platform/Deploy mirror
      ownership. Remove duplicates only after parity is demonstrated.
- [ ] Archive every previous repository except `.github` and
      `myota-docs` after no deployment, workflow, package, issue, or operator
      path depends on it.

Until these gates pass, `myota` is the integration candidate, while the
split-repository release remains active. Do not mark repository migration,
deployment, or archival complete.
