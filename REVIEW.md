# Cross-repository architecture review

**Reviewed:** 11 October 2026  
**Status:** Findings recorded; mitigation is proposed and has not been implemented.  
**Scope:** Read-only review of the MyOTA repositories on their main branches. No runtime code, database, cluster, or API behavior was changed for this review.

## Executive summary

The service split is understandable and the principal ownership boundaries are documented. Identity, Programme, Activity, Geodata, and Operations have separate owner repositories; Contracts owns the HTTP/event definitions; Deploy owns deployment; Platform and Deploy contain synchronized runtime/bootstrap copies. The review found two release-level inconsistencies that should be resolved before relying on affected paths:

1. The canonical OpenAPI file defines the destructive Geodata load-test cleanup path twice with different identifier constraints and descriptions. Duplicate YAML mapping keys have parser-dependent behavior.
2. Operations source has advanced beyond the Deploy and Platform runtime copies. In particular, the source owner imports a PostgreSQL/PostGIS sampler that is absent from both copies, and the Operations copies do not contain that sampler. The documentation records the sampler as live, so the source, image provenance, and live evidence need reconciliation.

Other confirmed cleanup includes retired NATS Operations paths still present in OpenAPI, duplicated runtime helper files, a large Geodata API module, remaining action-style API aliases, and stale architecture/status text. These findings do not establish a service outage or data loss. They do establish that source and contract parity cannot be assumed.

## Review method and limits

I reviewed repository READMEs, architecture and migration decisions, the central OpenAPI/event registry, service API modules and shared helpers, Deploy/Platform runtime copies and build entry points, the Admin route table, the shared GitHub workflow, and the active roadmap/status pages. Exact source paths are cited below.

This is a source and documentation review, not a production security audit. I did not execute tests, deploy images, call live service endpoints, inspect credentials, or inspect production data. Findings distinguish observed repository facts from risks/inferences. Treat runtime state claims as unverified unless a linked evidence record says otherwise.

## Source ownership and repository review

| Repository | Authoritative responsibility | Review notes |
|---|---|---|
| myota-contracts | Canonical OpenAPI, event schemas/registry, compatibility policy, client contract | The OpenAPI contract is broad and includes a duplicate destructive path definition plus retired Operations paths. Platform carries a byte-identical OpenAPI copy; it must remain a generated/synchronized mirror. See contracts/openapi.yaml and Platform contracts/openapi.yaml. |
| myota-identity-service | Identity, authentication, callsigns, roles and scopes | The service boundary is clear. Its common.py is byte-identical to Programme common.py. Its logging helper is byte-identical across all five services. These copies increase the number of places a shared security or telemetry fix must be released. |
| myota-programme-service | Programme configuration and shared category master data | The ownership is documented. Its common.py matches Identity exactly. The action-style update/archive and category assignment routes remain part of the API migration plan. |
| myota-activity-service | Activations, QSOs, imports, award evaluation/progress and issuance | The combined Activity/Awards process is an accepted service boundary in ADR-0005. The review found no evidence that splitting it into another service would reduce complexity safely. Keep the boundary unless measured ownership, load, or release coupling justifies a change. |
| myota-geodata-service | PostGIS entities, source provenance, imports, conflation, review and geodata workers | geodata.py is approximately 218 KB and contains a wide API surface. This is a maintainability hotspot, not proof of a runtime defect. Its Deploy and Platform copies differ from one another and from the owner source. |
| myota-operations-service | Domain-neutral storage inspection, metrics, history and Grafana identity | Current source has no NATS inspection routes. It imports postgres_observability.py and starts the database sampler. That module and implementation are absent from Deploy and Platform service copies. |
| myota-deploy | Compose, Helm, deployment orchestration, runtime image build and deployment-owned provisioning | Dockerfile copies the repository into the runtime image; the image workflow builds from this repository. The services/ files therefore affect built runtime images and need a deterministic source-sync gate. |
| myota-platform | Integration bootstrap and synchronized runtime, contract and migration mirrors | It duplicates Deploy service code and contract/deployment assets. It is not a domain source owner. At least the Geodata runtime copy differs from Deploy as well as from the Geodata source. |
| myota-web | Participant-facing web experience | The separate participant boundary is intentional. This review did not find evidence for merging it with administration. API migration must include its actual client calls before retiring aliases. |
| myota-admin-web | Authenticated administration UI | The current router contains the storage view but no JetStream status route. This confirms that older repository-map text about an active NATS page is stale. |
| myota-docs | Architecture, ADRs, evidence, plans and status indexes | Some architecture and status text still describes already-retired NATS monitoring. The open phase gates are summarized below and linked from the indexes. |
| .github | Organization profile and shared quality workflow | The shared Python workflow runs Ruff formatting and lint only. It does not detect duplicate OpenAPI/YAML keys or owner-to-mirror drift. Add narrowly scoped repository consistency checks in the mitigation phases. |

The exact main-branch snapshots reviewed include Identity 35e8ed3, Programme 2d7853b, Activity 11f76ae, Geodata aa82924, Operations 863595a, Contracts c55ebf7, Deploy 85d775c, Platform a0d7244, and Docs afbe3ca. The review also inspected the current main contents of both web repositories and .github. Re-run the source and mirror comparisons before implementation if any main branch has advanced.

## Findings

### A-01 — Duplicate destructive OpenAPI path definition

**Severity: High; contract correctness and destructive-operation safety. Confirmed.**

The canonical contract at myota-contracts/contracts/openapi.yaml defines DELETE /v1/geodata/load-test-runs/{testRunId} twice (around lines 502 and 1329). The first definition permits any testRunId with length 8–64 and describes exact confirmation. The second restricts IDs to the lt- pattern with a different length bound, has a distinct summary/description, and contains the operationId. These are conflicting definitions for one YAML mapping key. Parsers may silently retain one definition, so generated clients and validation can depend on parser behavior.

The Geodata service owns the handler at myota-geodata-service/geodata.py. The contract says cleanup refuses active runs or linked activity/award impact, but the reviewed duplicate itself is enough to require one canonical schema and a fail-closed contract check. Do not broaden the identifier pattern or weaken the confirmation rule while removing the duplicate.

**Mitigation:** In Phase 0, compare the handler and both contract entries, select the single intended request/response contract, add duplicate-key rejection to contract validation, and verify generated client output. Preserve the existing authorization, feature-flag, production-acknowledgement, exact-confirmation, and linked-data protections.

### A-02 — Operations source and runtime mirrors disagree

**Severity: High; release provenance and operational visibility. Confirmed source drift; live impact is unverified.**

The source owner myota-operations-service/operations.py imports poll_postgres from myota-operations-service/postgres_observability.py, starts a sampler thread, and joins it on shutdown. That module exists in the owner repository. The current copies myota-deploy/services/operations.py and myota-platform/services/operations.py do not import or start that sampler, and neither mirror contains services/postgres_observability.py. Deploy's Dockerfile copies its own repository into the image and its build-and-publish-images workflow builds from that repository. Therefore, a clean image build from current Deploy main cannot include the source owner's PostgreSQL fallback implementation. The documentation evidence currently says the fallback is active.

This is a concrete source/image lineage inconsistency. It does not by itself prove which image is currently running. Before claiming the database metrics are deployed, reconcile the image digest and source commit with the runtime artifact.

The same general drift appears in Geodata: myota-geodata-service/geodata.py is larger and newer than myota-deploy/services/geodata.py; myota-platform/services/geodata.py differs from both. Deploy and Platform versions also differ from each other in the entity-deletion path. Their synchronized-mirror status is documented in docs/architecture/repository-map.md, but there is no shared ownership boundary inside these duplicated files that makes independent edits safe.

**Mitigation:** Phase 1 establishes owner-to-mirror synchronization and an image provenance check. Do not manually patch only one copy. Keep Geodata and Operations owner repositories authoritative, regenerate or copy the relevant files into Deploy and Platform from the owner commit, and fail CI when expected synchronized paths differ.

### A-03 — Retired NATS API is still advertised by OpenAPI

**Severity: Medium; API/contract consistency. Confirmed.**

The current Operations handler exposes observability-session and object-storage routes, but not /v1/operations/jetstream or /v1/operations/jetstream/snapshots. The Admin router has no JetStream page. The Surveyor consolidation plan records the NATS sampler, history table, and Admin page as retired. However, myota-contracts/contracts/openapi.yaml still declares both JetStream endpoints.

This leaves the canonical contract describing endpoints that the current owner no longer implements. It can produce unusable generated client methods and mislead API consumers. Confirm whether any independently released client still depends on them, then remove or mark the contract operations obsolete in the contract owner and update the Platform contract mirror.

### A-04 — Shared runtime helpers are copied across service repositories

**Severity: Medium; maintenance/security fix propagation. Confirmed duplication; extraction choice is proposed.**

All five service repositories contain byte-identical myota_logging.py files (the same blob SHA). Identity and Programme common.py are byte-identical; Geodata and Operations common.py are byte-identical; Activity has a separate version. These files contain shared HTTP, persistence, authentication, and telemetry primitives. Copying them makes a shared fix a multi-repository release task and allows the copies to diverge.

Independent service builds are a valid constraint, so extracting a shared library is not automatically simpler. Phase 2 should first classify functions by semantics and release coupling. Then choose one supported approach: a versioned internal runtime package with pinned dependencies, or a generated/synchronized source module with a strict parity check. Do not centralize domain-specific authorization or persistence behavior simply because it appears in a common.py file.

### A-05 — Geodata API module is a complexity hotspot

**Severity: Medium; maintainability risk, not an asserted correctness defect.**

myota-geodata-service/geodata.py is approximately 218 KB and coordinates a broad set of import, entity, review, deletion, location and API operations. The repository also contains specialized modules such as geodata_import_worker.py, relational_state.py, relational_queries.py, import_adapters.py, and geodata_retention_worker.py. The large API module makes ownership, change review, and route coverage harder to reason about. The exact module size alone is not a reason to create another service.

Phase 3 proposes a behavior-preserving module split around existing cohesive responsibilities. Keep transactions, authorization, idempotency, outbox writes, and database ownership in Geodata. Use the existing route/handler contract as the compatibility boundary and avoid introducing new service-to-service calls.

### A-06 — API aliases and action-style endpoints remain

**Severity: Medium; known migration work, not an unexpected defect.**

The canonical API contains action-shaped routes including Programme update/archive and entity-type assign/unassign, Identity admin update/deactivate, and Geodata entity delete. The docs/domain/api/rest-consolidation-plan.md records Phases 0–4 as implemented and Phase 5 as planned. It explicitly requires usage evidence, client migration, and sunset communication before alias removal.

The endpoint surface is large (145 path entries in the current OpenAPI file), spanning five bounded services. Some actions are appropriate commands, such as review, publish, confirm, and job start; do not mechanically convert every action into CRUD. Complete the existing Phase 5 plan by operation and client, retire only true aliases after usage reaches zero or owners migrate, and remove redundant contract copies only when generation/checks replace them.

### A-07 — Architecture documentation contains stale NATS ownership/status text

**Severity: Medium; documentation drift. Confirmed and corrected where directly encountered in this review.**

The current repository map says the Operations and Admin NATS inspection paths remain current until a rollout/overlap gate passes. The Surveyor plan and current Admin router say those paths are retired. The repository map also describes the retired Admin status page as current. The migration plan records Phase 0–6 as complete within its evidence limits. These statements cannot all describe the current design.

The architecture map is corrected in this documentation change to state that Surveyor owns broker monitoring and that Operations has no NATS inspection connection. The broader runtime findings remain open until their source owners and mirrors are reconciled.

## Open work and phase status

These items are taken from the current linked roadmaps. They are not marked complete by this review.

| Workstream | Current status in its source plan | Remaining work |
|---|---|---|
| NATS event/work migration | Phases 0–6 complete within recorded evidence bounds; the Phase 5 24-hour observation was waived and closed early, not completed as a full 24-hour observation. | No migration phase is recorded as open. PostgreSQL remains authoritative and off-node recovery is deferred. |
| NATS Surveyor consolidation | Phases 0–2 complete; current plan records retired Operations/Admin/Geodata monitoring paths and the history-table migration. | No plan phase is recorded as open. |
| Geodata horizontal scaling | Phases 0–4 complete for their stated evidence limits; Phase 5 remains open. | Performance acceptance, independent-row consistency, node/storage failure, canary, rollback, and replica-range expansion evidence remain open in docs/geodata/horizontal-scaling-roadmap.md. |
| Structured logging / database telemetry | Implementation is in progress; do not treat source/configuration checkboxes as overall acceptance. | Helm/Fleet release-state agreement, authenticated Grafana navigation, representative request-to-worker correlation, Loki prohibited-field audit, Compose parity, and bounded retention/capacity evidence remain open. The latest evidence says Operations fallback DB metrics are active; direct Collector PostgreSQL receiver remains disabled. |
| REST API consolidation | Phases 0–4 implemented; Phase 5 planned. | Usage and client migration, sunset notice, safe alias removal, duplicate contract copy cleanup, and migration guide remain open. |
| Programme configuration | Audit baseline; implementation pending. | Ownership/lifecycle, locale/content, jurisdiction/approver, eligibility, and privacy/identity configuration remain roadmap work. |
| Identity and Programme metrics | Observability overview records a live scrape gap for both services. | Resolve endpoint/scrape configuration and verify live metrics; do not infer API unavailability from /metrics failures. |

The root To do and Prioritized backlog indexes are updated to include this review. The active Work in progress index continues to point at the detailed rollout gates; the review does not reclassify unstarted architecture cleanup as implementation in progress.

## Proposed phased mitigation plan

Every phase below is a separate implementation prompt. Each prompt is self-contained, names source owners and mirrors, and is intentionally limited to one phase. The prompts are proposals for future work; none means implementation has started.

### Phase 0 — Reconcile API contracts and live route ownership

**Repositories:** myota-contracts (authority), myota-geodata-service, myota-operations-service, myota-admin-web, myota-web, myota-platform (contract mirror), myota-docs, and .github.

**Tasks**

- [ ] Read this review and docs/domain/api/rest-consolidation-plan.md before editing.
- [ ] Compare every OpenAPI path/method touched in this phase with its service-owned handler and current web/client usage.
- [ ] Reduce the duplicate Geodata load-test cleanup path to one fail-closed operation definition that matches the handler’s authorization and confirmation behavior.
- [ ] Remove duplicate YAML-key ambiguity from the contract validation path and add a focused CI guard for duplicate mapping keys.
- [ ] Resolve the two retired Operations JetStream contract entries after checking client references; keep current storage and Grafana identity operations.
- [ ] Record any remaining intentional contract/runtime exceptions with owner and retirement condition.

**Exit criteria**

- [ ] OpenAPI parses with duplicate mapping keys rejected, not silently overwritten.
- [ ] Each in-scope operation has one owner handler or an explicit documented exception.
- [ ] The cleanup route has one request schema and preserves existing destructive-operation guards.
- [ ] No current Admin or web client calls a retired Operations JetStream route.
- [ ] Contracts CI and focused service/client checks pass; exact commands and results are recorded.

**Copyable implementation prompt**

~~~text
Implement Phase 0 of the cross-repository architecture mitigation plan in
myota-docs/REVIEW.md. Read that review and
myota-docs/docs/domain/api/rest-consolidation-plan.md first.

Use myota-contracts as the OpenAPI source owner. Inspect the corresponding
Geodata and Operations handlers, Admin and participant client references, and
the synchronized OpenAPI copy in myota-platform. Do not edit the contract
mirror independently.

Resolve the duplicate DELETE
/v1/geodata/load-test-runs/{testRunId} definition in the canonical OpenAPI
file by matching one final schema to the Geodata handler. Preserve existing
authorization, explicit enablement, production cleanup acknowledgement,
exact confirmation, and linked-activity/award safety checks. Remove the
retired Operations JetStream contract paths only after checking all current
client references; retain live storage and Grafana identity paths.

Add a focused contract check that rejects duplicate YAML mapping keys and
verifies the affected operation-to-handler/client mapping. Do not change
unrelated routes or service behavior. Run only focused checks for this phase,
record exact commands/results, update the Platform mirror through its normal
sync path, and report changed repositories, remaining evidence, and exit
criteria. Mark a checkbox complete only when its evidence is recorded.
~~~

### Phase 1 — Restore owner-to-mirror and image-source parity

**Repositories:** the five service owners (Identity, Programme, Activity, Geodata, Operations), myota-deploy (runtime image/deployment owner), myota-platform (bootstrap mirror), myota-contracts, and .github.

**Tasks**

- [ ] Define a path-level source-owner map for every mirrored runtime, migration, registry, schema, and contract artifact.
- [ ] Synchronize Operations and Geodata mirrors from their owner repositories, including postgres_observability.py where the owner module is required.
- [ ] Determine whether Deploy and Platform need identical service snapshots or have an explicitly versioned divergence; eliminate undocumented differences.
- [ ] Add a reproducible owner-to-mirror audit to CI and fail on unapproved drift.
- [ ] Record image build context, source commit, and digest together so a running digest can be traced to its source tree.

**Exit criteria**

- [ ] Every mirrored path has one named source owner and an automated sync/check.
- [ ] Relevant owner and mirror content matches, or each intentional difference has a reason, owner, and expiry/review condition.
- [ ] A clean Deploy image build contains all modules imported by its runtime entrypoints.
- [ ] Runtime evidence maps the deployed digest to the expected Deploy commit and source-owner revisions.
- [ ] Deploy, Platform, and owner-repository CI checks pass; no repository is edited as an independent authority.

**Copyable implementation prompt**

~~~text
Implement Phase 1 of the cross-repository architecture mitigation plan in
myota-docs/REVIEW.md. Read the review and current repository-map documentation
first. Treat Identity, Programme, Activity, Geodata, Operations, and Contracts
as their respective source owners. Treat myota-deploy as the deployment/image
owner and myota-platform as the integration/bootstrap mirror. Do not make a
mirror an independent source of truth.

Build a path-level synchronization map, then synchronize all in-scope files
from their owner revisions. At minimum reconcile myota-operations-service/
operations.py and postgres_observability.py with myota-deploy/services and
myota-platform/services, and reconcile the differing Geodata API copies.
Check OpenAPI, event registry, schemas, and mirrored migrations using their
documented owners.

Add a repeatable CI drift audit and capture the source commit used by a
deployment image. Preserve service boundaries, API behavior, authorization,
database ownership, and migration ordering. Do not deploy or mutate the live
cluster as part of this phase unless separately authorized by the roadmap.
Run focused build and synchronization checks, record exact results and image
provenance, update the review/indexes, and only check exit criteria supported
by evidence.
~~~

### Phase 2 — Reduce copied shared-runtime code safely

**Repositories:** Identity, Programme, Activity, Geodata, Operations, Deploy, Platform, and .github. Service repositories remain authoritative for domain behavior.

**Tasks**

- [ ] Inventory and classify functions in each common.py and myota_logging.py copy by behavior, security sensitivity, dependency, and release cadence.
- [ ] Choose either a pinned shared runtime package or generated source synchronization based on actual build constraints.
- [ ] Keep domain-specific authorization, database repositories, and service policy in their owning service.
- [ ] Migrate one helper family at a time; keep compatibility wrappers only while callers move.
- [ ] Remove duplicate implementations after call-site and mirror parity evidence.

**Exit criteria**

- [ ] Every shared function has one maintained implementation or a documented reason for an independent implementation.
- [ ] The chosen release/sync mechanism works in isolated service builds and in Deploy/Platform builds.
- [ ] Authentication, redaction, request/correlation behavior, persistence transactions, and error mapping retain their existing semantics.
- [ ] Focused service checks and source/mirror audit pass; no copy is silently stale.

**Copyable implementation prompt**

~~~text
Implement Phase 2 of the cross-repository architecture mitigation plan in
myota-docs/REVIEW.md. Read the review and Phase 1 source/mirror result first.
The five service repositories own their service behavior; Deploy and Platform
are build/bootstrap mirrors.

Compare common.py and myota_logging.py across Identity, Programme, Activity,
Geodata, and Operations. Classify each function before changing it, especially
authentication/token handling, transaction behavior, error mapping, trace
propagation, and privacy redaction. Select the lowest-complexity sustainable
mechanism: a versioned pinned runtime package only if independent builds can
consume it reliably, otherwise a generated/synchronized module with drift
checks. Do not move service-specific authorization or persistence policy into
a generic layer.

Migrate one helper family at a time, preserving public imports until callers
are moved. Keep owner repositories authoritative and update Deploy/Platform
only through the synchronization mechanism. Run focused checks for each
affected service and the mirror audit. Record the option chosen, tradeoffs,
changed files, exact results, and remaining duplicate code. Mark exit criteria
only with evidence.
~~~

### Phase 3 — Simplify service implementation hotspots without changing boundaries

**Repositories:** myota-geodata-service first; myota-activity-service and myota-programme-service only for ownership documentation; myota-contracts and myota-docs for contract/decision updates; then Deploy and Platform mirrors.

**Tasks**

- [ ] Map Geodata routes to cohesive modules already present for imports, entities, review, deletion, location, and query behavior.
- [ ] Split geodata.py into cohesive modules without creating new services or changing route semantics.
- [ ] Clarify the boundary between Programme-owned policy/configuration and Activity-owned award definitions, evaluation, progress, requests, and issuance.
- [ ] Use measured change coupling and runtime load before proposing a new service or database.
- [ ] Synchronize mirrors only after owner changes are complete.

**Exit criteria**

- [ ] Every Geodata route and background operation has one obvious owning module and retains the same API contract.
- [ ] Authorization, transaction/outbox boundaries, idempotency, leases, and database ownership remain unchanged.
- [ ] Programme versus Activity award/configuration ownership is documented with concrete data and API examples.
- [ ] Focused regression checks and migration/API contract checks pass; no new service or database was added without a separate accepted decision.

**Copyable implementation prompt**

~~~text
Implement Phase 3 of the cross-repository architecture mitigation plan in
myota-docs/REVIEW.md. Read the review, ADR-0005, the repository map, and the
Geodata and Activity service designs before editing. Geodata owns geospatial
entities and import/review workflows. Activity owns activations, QSOs, award
evaluation/progress, requests, rendering, and issuance. Programme owns
programme policy and configuration.

First produce a route-to-module and worker-to-module map for
myota-geodata-service/geodata.py and its existing specialized modules. Then
split the Geodata API implementation along cohesive existing responsibilities
without changing endpoints, authorization, transaction boundaries, outbox
writes, idempotency, recovery, or physical database ownership. Do not create
another service or database in this phase.

Clarify the Programme/Activity award boundary using existing data and API
examples. If evidence does not justify moving ownership, preserve the current
boundary and document it. Update Deploy/Platform mirrors from the owner
repositories after source changes. Run focused behavior and contract checks,
record exact results and any residual hotspots, and mark only verified exit
criteria complete.
~~~

### Phase 4 — Finish API alias retirement and consolidate the architecture record

**Repositories:** myota-contracts, Identity, Programme, Activity, Geodata, Operations, both web clients, Deploy, Platform, .github, and myota-docs.

**Tasks**

- [ ] Continue from the existing REST consolidation Phase 5 rather than creating a competing API plan.
- [ ] Measure remaining alias usage and enumerate callers by repository/version.
- [ ] Publish compatibility and sunset guidance before removing aliases with external use.
- [ ] Remove only aliases proven unused or explicitly migrated; preserve intentional command endpoints.
- [ ] Remove duplicate contract copies only after the sync/generation checks from Phase 1 are established.
- [ ] Reconcile repository maps, architecture diagrams, To do, Work in progress, backlog, and release documentation with evidence.

**Exit criteria**

- [ ] Every removed route has zero observed use over the plan's stated measurement window or recorded migration acceptance from its client owner.
- [ ] Intentional command/job routes remain documented; no blanket CRUD conversion occurred.
- [ ] Canonical contract, generated client, service handlers, and mirrors agree.
- [ ] The open work indexes match their source roadmaps and name each remaining gate.
- [ ] The full mitigation plan has a final evidence summary; no item is marked complete based only on code presence.

**Copyable implementation prompt**

~~~text
Implement Phase 4 of the cross-repository architecture mitigation plan in
myota-docs/REVIEW.md. Read the Phase 0–3 evidence and
docs/domain/api/rest-consolidation-plan.md first. Continue that plan's Phase 5;
do not invent a second deprecation policy.

Use myota-contracts as the canonical API source, each service repository as
handler owner, myota-web and myota-admin-web as client owners, myota-deploy as
deployment owner, and myota-platform as a synchronized bootstrap mirror. Use
usage evidence to identify remaining aliases and clients. Publish migration and
sunset guidance before removing any route that may have external use. Preserve
intentional command/job routes such as review, publish, confirm, and job start.

Remove contract copies only after the Phase 1 generation/sync gate is working.
Reconcile the repository map, diagrams, this review, To do, Work in progress,
prioritized backlog, and changes/timeline records with the implementation
evidence. Run focused contract/client/service checks, report exact results and
remaining gaps, and do not mark any checkbox complete without evidence.
~~~

## Immediate documentation corrections made with this review

- Updated docs/architecture/repository-map.md to show NATS Surveyor as the broker monitoring owner and to record that Operations/Admin JetStream inspection has been retired.
- Added this review to the documentation and architecture indexes, the To do list, and the prioritized backlog.
- Added a dated entry to changes.md.
- Updated the organization profile to link this review and to distinguish the active Operations database sampler from the disabled direct Collector PostgreSQL receiver.

These are documentation updates only. No application or deployment code, API behavior, schema, database, or cluster state was changed.