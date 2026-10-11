# Cross-repository architecture review

**Reviewed:** 2026-10-11  
**Status:** Phases 1–4 implemented; the separate Phase 0 duplicate-YAML-key
safeguard remains open.  
**Scope:** Identity, Programme, Activity, Geodata, Operations, Contracts, Deploy,
Platform, both first-party web clients, Docs, and .github.

## Executive summary

The four requested mitigation phases are implemented and recorded:

1. Owner-to-mirror parity now has a file/tree manifest and a deterministic
   Contracts CI audit. Contracts and migration artifacts used by Deploy and
   Platform are included.
2. Shared logging has one canonical source with a synchronization script.
   Service HTTP, persistence, upload, authorization, and metrics adapters remain
   in service-owned common modules.
3. Geodata route registration is separated into route groups. Existing
   import/worker/repository ownership and the Programme/Activity award boundary
   remain unchanged.
4. First-party clients and the current contract use the canonical resources.
   Unused compatibility-only API aliases and their runtime deprecation
   machinery have been removed. Audited domain commands remain explicit.

The current route inventory has 132 OpenAPI operations and 170 registered
routes, with no missing operations or duplicate operation identifiers.
Identity, Programme, Activity, Geodata, Operations, Deploy, Platform, and
Contracts checks pass at the revisions linked in the
[phase evidence record](docs/architecture/evidence/review-phases-1-4-2026-10-11.md).

One independent contract-safety issue remains open: the route inventory does
not reject duplicate YAML mapping keys. The destructive Geodata cleanup
operation still needs a duplicate-key-aware parser check. It is tracked as
Phase 0 and in the To do index; this review does not mark it complete.

## Ownership decisions

| Repository | Authority |
| --- | --- |
| myota-contracts | Canonical OpenAPI, event schemas, registry, and typed client facades. |
| Identity, Programme, Activity, Geodata, Operations | Service-owned handlers, persistence, authorization, events, and migrations. |
| myota-deploy | Runtime image, Compose, Helm, migration orchestration, and deployment-owned provisioning. |
| myota-platform | Integration/bootstrap copy; not an independent domain authority. |
| myota-web and myota-admin-web | First-party client owners. |
| .github | Organization profile and reusable quality workflows. |
| myota-docs | Architecture, decisions, evidence, and status indexes. |

## Findings and current disposition

### A-01 — Duplicate destructive OpenAPI mapping

**Severity:** High. **Status:** open in Phase 0.

The canonical OpenAPI file contains duplicate mapping-key definitions for the
Geodata load-test cleanup operation. The definitions disagree on identifier
constraints and descriptions. Ordinary YAML parsing can silently retain one
value. The current route inventory checks methods, paths, operation IDs, and
runtime registrations, but does not reject duplicate YAML mapping keys.

**Required follow-up:** add duplicate-key rejection to the Contracts validation
path, compare the single contract definition with the Geodata handler, and
record the exact parser/test result. Preserve the current handler's
authorization, explicit enablement, production acknowledgement, exact
confirmation, and linked-data safeguards.

### A-02 — Operations and Geodata mirror drift

**Severity:** High. **Status:** implementation complete; deployment evidence is
recorded in the phase evidence page.

Contracts now maps 44 source-owned files and 92 file mirrors, plus five exact
tree mirrors covering event schemas and Activity/Geodata migrations in Deploy
and Platform. The map includes Operations' PostgreSQL sampler and Geodata
runtime modules. The audit compares file bytes, directory membership, and
directory contents. Deploy remains the runtime image owner; Platform remains
the integration mirror.

### A-03 — Retired Operations broker-inspection contract

**Severity:** Medium. **Status:** resolved.

The contract entries that had no Operations handler or first-party client were
removed. Broker monitoring is owned by NATS Surveyor and Grafana. Operations
remains responsible for its domain-neutral storage and database observations.

### A-04 — Copied shared runtime helpers

**Severity:** Medium. **Status:** shared logging source established; adapter
copies remain documented.

The logging helper now has one Platform-owned canonical source and a
deterministic synchronization script for all service and deployment build
roots. The service-local common modules combine different HTTP, persistence,
upload, authentication, error, and metrics adapters. They remain with their
service owners to preserve independent service builds and domain boundaries;
no runtime package dependency was introduced. This is a conscious residual
duplication, not a claim that all common modules have one implementation.

### A-05 — Geodata implementation hotspot

**Severity:** Medium. **Status:** route organization improved; handler module
remains a known maintainability residual.

Geodata route registration is organized in
myota-geodata-service/geodata_routes.py into catalogue, import, and job groups.
Existing specialized import, worker, state, and query modules retain their
roles. Geodata remains the only spatial data owner. No additional service or
database was justified. The larger handler module is retained until a
behavior-preserving boundary with isolated state and regression coverage is
measurable.

### A-06 — API aliases and action-shaped operations

**Severity:** Medium. **Status:** resolved for the current pre-live clients and
contract.

First-party Admin and public client sources were checked for calls to the
compatibility-only aliases. None were found. Those aliases were removed from
service route registries, canonical OpenAPI, and client coverage. Explicit
audited lifecycle commands, job resources, and the Geodata/Activity deletion
workflow remain. The route inventory verifies current contract/handler
agreement.

### A-07 — Architecture and status drift

**Severity:** Medium. **Status:** corrected in this change.

The API plan, phase records, diagrams, work indexes, prioritized backlog,
repository map, implementation timeline, changes page, root README, and
organization profile now describe current API ownership, retired alias
behavior, and remaining work. Operating guidance no longer instructs readers
to retain removed aliases or use removed API routes.

## Phase 1 — Restore owner-to-mirror and image-source parity

**Status:** complete.

- [x] Map selected synchronized runtime files, canonical contract files,
  event-schema tree, and exact migration trees consumed by Deploy and Platform.
- [x] Synchronize Operations and Geodata owner files into Deploy and Platform.
- [x] Document Contracts, service, Deploy, and Platform ownership.
- [x] Add a CI audit for file bytes and exact tree contents.
- [x] Run owner quality/image workflows and the Deploy image workflow; record
  current image digests, Fleet commit, Helm revision, and readiness.

**Evidence:** the source map and passing audit are linked in the phase evidence
page. The route inventory, OpenAPI, event registry, event schemas, and migration
trees pass the Contracts checks. Current image revisions and cluster readiness
are recorded after Fleet reconciliation.

## Phase 2 — Reduce copied shared-runtime code safely

**Status:** complete with a documented service-local adapter decision.

- [x] Establish Platform as the single source for the shared logging module.
- [x] Synchronize it to the five services and both Deploy/Platform runtime roots.
- [x] Preserve authentication, persistence, redaction, correlation, and error
  policy within the owning service adapters.
- [x] Document why common.py stays service-local and avoid adding a package
  dependency to independent service builds.
- [x] Run the owner builds and source/mirror audit.

The decision intentionally removes duplicate maintenance for the logging
helper. It keeps the broader HTTP/persistence adapter files local because they
are service-specific integration code. Their remaining similarity is recorded
as a tradeoff.

## Phase 3 — Simplify the Geodata hotspot without moving service boundaries

**Status:** complete for the accepted route/ownership scope; the large handler
module remains an explicit residual.

- [x] Move Geodata route declarations into cohesive catalogue, import, and job
  groups.
- [x] Keep handlers, worker state, authorization, transactions, outbox writes,
  idempotency, recovery, and database ownership in Geodata.
- [x] Document Programme-owned policy versus Activity-owned award execution.
- [x] Keep Deploy and Platform route copies synchronized.
- [x] Run Geodata quality/image checks and contract route inventory checks.

A deeper handler split is deferred because no low-risk independently testable
boundary was identified in this phase. The review records this decision and
does not claim the handler module is small.

## Phase 4 — Retire unused API aliases and reconcile documentation

**Status:** complete.

- [x] Search first-party client sources and current service/contract registries.
- [x] Remove aliases with no first-party callers in the pre-live system.
- [x] Preserve audited domain commands and asynchronous job boundaries.
- [x] Remove alias-specific response headers, route registries, and metrics.
- [x] Align canonical OpenAPI, route inventory, service handlers, and mirrors.
- [x] Update diagrams, API pages, work indexes, backlog, changes, timeline,
  README links, and .github profile links.
- [x] Record current CI results and deployment evidence.

There is no user migration or runtime observation gate for these removed
aliases in this pre-live system.

## Remaining Phase 0 work

- [ ] Add duplicate-YAML-mapping-key rejection to the Contracts validation path.
- [ ] Verify the single destructive Geodata cleanup definition against its handler and synchronized OpenAPI mirror.
- [ ] Record the exact parser and CI result before closing this item.

## Remaining Phase 0 prompt

~~~text
Complete the remaining Phase 0 contract-safety item in myota-docs/REVIEW.md.
Read this review, the API contract freeze record, and the current Geodata
cleanup handler. Treat myota-contracts as the OpenAPI authority and the
Platform OpenAPI file as a synchronized mirror.

Add a focused YAML validation check that rejects duplicate mapping keys.
Reduce the current Geodata cleanup operation to one OpenAPI definition that
matches its owner handler. Preserve authorization, explicit enablement,
production acknowledgement, exact confirmation, and linked-data safeguards.
Regenerate the route inventory and verify the OpenAPI mirror.

Run the focused Contracts check and report exact file changes and results.
Update the Phase 0 checkbox and this review only when the duplicate-key check,
handler comparison, and mirror evidence pass.
~~~

## Verification boundary

The work changed source code, Contracts, synchronized runtime files,
deployment inputs, and documentation. CI results and deployed cluster state
are linked from the [phase evidence record](docs/architecture/evidence/review-phases-1-4-2026-10-11.md).
The remaining duplicate-key safeguard is not implied complete by a passing
route inventory.
