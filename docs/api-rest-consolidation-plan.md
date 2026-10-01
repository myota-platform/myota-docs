# REST API consolidation plan

Status: proposal only — no runtime code changed  
Reviewed: 2026-09-30
Owner: myota-contracts with the affected service repositories

This review compares the current OpenAPI documents and route registries in the
MyOTA repositories. It identifies redundant action-style routes and proposes
a more consistent REST resource model. It is a design and migration plan,
not an authorization to change the API yet.

## Executive summary

The current API mixes resource-oriented routes, field-specific mutation
routes, and action routes such as update, archive, publish, close, delete,
review, run, and issue.

The action style is not automatically wrong. Authentication, destructive
cross-service deletion, audited lifecycle transitions, and asynchronous work
are meaningful domain commands. They should not be disguised as ordinary
PATCH operations. The recommendation is selective consolidation:

- use PATCH, PUT, and collection subresources for ordinary resource mutations;
- use a resource representing a review, upload, export, deletion job,
  recalculation job, or render job for asynchronous/audited workflows; and
- retain authentication and security-sensitive commands as explicit actions.

## Inventory findings

### Contract copies can drift

The organization currently contains:

- the canonical contract candidate in myota-contracts/contracts/openapi.yaml;
- a duplicate root-level myota-contracts/openapi.yaml; and
- an integration copy in myota-platform/contracts/openapi.yaml.

The service route registries do not all expose exactly the same surface as the
canonical contract. Before changing paths, establish one canonical document
and make the other copies generated or CI-checked mirrors. The consolidations
below apply to the canonical contract and owning service.

### Ordinary updates use POST action suffixes

Account editing, programme editing, entity metadata editing, and award
lifecycle changes are mostly mutations of existing resources. Their current
update, name, location, entity-type, submit, publish, and similar routes make
generic clients and compatibility tooling harder than necessary.

### Long-running operations need job resources

Refresh, deletion/cascade, statistics rebuild, award recalculation, and PDF
rendering are not ordinary synchronous updates. They should create explicit
job resources that can be polled, retried, audited, and correlated with events.

## Current contract and route baseline

The following is the current implementation baseline as of 2026-09-30. The
activity and awards APIs are served by the same `myota-activity-service` HTTP
process and port (`8004` locally), but remain separate resource namespaces and
data boundaries.

| Service area | Current implemented surface | Current contract/route note |
| --- | --- | --- |
| Identity and administration | `/v1/identity/auth/*`, `/v1/identity/me`, account/callsign/role/security-event routes under `/v1/identity/` | Authentication commands remain explicit. Account, role, callsign, and export/deactivation routes are candidates for resource aliases. |
| Programme configuration | `/v1/programmes`, `/v1/programmes/{slug}`, `/update`, `/archive`, policy, content, policy-draft, and entity-type assignment routes | Programme rules, awards, content, and category membership remain programme-owned; no POTA rules are implied by the transport model. |
| Geodata catalogue | `/v1/geodata/entities`, `{entityId}`, `/audit`, `/bbox`, `/tiles/{z}/{x}/{y}`, `/adapters`, `/location-options`, and `/conflation` | Entity listing already supports programme-independent filters, multi-status values, location filters, pagination, and map bounds. |
| Geodata intake | `POST /v1/geodata/imports`, `/imports/manual`, `/imports/upload`; run, candidate, validation, processing, and finalization routes | Imports are programme-independent. The staged lifecycle is `UPLOAD_PENDING`/`QUEUED` → `PROCESSING` → `PREPROCESSED` or `PREPROCESSED_WITH_ERRORS`; only validated records enter the promotion queue. |
| Geodata review/editing | `POST /entities/{id}/review`, `/status`, `/geometry`, `/geometry-type`, `/location`, `/entity-type`, `/name`, `/delete` | Candidate review and entity management are separate UI workflows, but the API still has several overlapping action routes. Approved entities may only become `RETIRED`. |
| Activity | `/v1/activations`, activation QSOs and batch QSOs, close, ADIF imports, QSO corrections, public history/leaderboards/results, statistics, notifications | ADIF and batch ingestion are asynchronous/high-volume candidates for ingestion resources; close and correction review are audited commands. |
| Awards | `/v1/awards`, assets, submit/review/publish/retire, evaluate/progress, requests, issuances, render, download | Awards are programme-owned and versioned. Asset uploads, evaluation, recalculation, rendering, and issuance should expose durable job/resource state. |
| Cross-service deletion | Activity deletion-impact and cascade-delete routes followed by geodata deletion | This is intentionally a coordinated workflow, not a simple entity delete. It invalidates linked QSOs and may trigger award recalculation. |

### Latest contract gaps

The canonical OpenAPI contract now describes the staged geodata import queue,
candidate validation and promotion targets, location hierarchy, geometry-type
conversion, activity deletion impact, programme-owned awards, award assets,
issuance, and certificate rendering. It still needs reconciliation with the
live registries in these areas:

- Add or document the live geodata `status`, `geometry`, `audit`, `bbox`, tile,
  adapter, schedule, and conflation operations consistently in the canonical
  contract and generated mirrors.
- Document the multipart upload representation and the 1 GiB deployment limit;
  the JSON/Base64 upload remains a compatibility path, not the preferred
  browser path.
- Add the live activity ADIF, batch-QSO, correction, public-results,
  statistics, notifications, and deletion workflow operations to the same
  contract revision.
- Add identity primary-callsign, callsign retirement, evidence, OIDC mapping,
  and account role-assignment details where they are currently only present in
  the service registry.
- Keep the root contract copy and the platform integration copy generated or
  CI-checked against `myota-contracts/contracts/openapi.yaml`; do not manually
  maintain divergent route definitions.

The route inventory above is evidence for the consolidation work. It is not a
request to remove the current compatibility routes before aliases and client
migrations exist.

## Proposed endpoint consolidation

The table lists the proposed target and the current routes it would replace or
alias. This is a migration target, not a list of routes to remove immediately.

| Area | Proposed REST endpoint(s) | Current routes consolidated | Recommendation |
| --- | --- | --- | --- |
| Identity account editing | PATCH /v1/identity/accounts/{accountId} | POST /v1/identity/admin/accounts/{accountId}/update | Move editable account fields to the account resource; retain admin authorization. |
| Account deactivation | PATCH /v1/identity/accounts/{accountId} with status=DEACTIVATED | POST /v1/identity/accounts/{accountId}/deactivate | Treat deactivation as a state update with audit and session revocation. |
| Administrative role editing | PATCH /v1/identity/roles/{roleCode} | POST /v1/identity/admin/roles/{roleCode}/update | Use the role resource and protect built-in roles in policy. |
| Account role assignments | PUT /v1/identity/accounts/{accountId}/role-assignments | POST /v1/identity/accounts/{accountId}/roles | Replace the assignment set atomically; retain GET on the collection. |
| Callsign lifecycle | PATCH /v1/identity/accounts/{accountId}/callsigns/{callsignId} | POST .../verify and POST .../retire | Use audited lifecycle fields; evidence remains a created subresource. |
| Primary callsign | PUT /v1/identity/accounts/{accountId}/primary-callsign | POST /v1/identity/accounts/{accountId}/primary-callsign | Keep the relationship explicit, but make it idempotent PUT. |
| Account export | POST /v1/identity/accounts/{accountId}/exports; GET /v1/identity/exports/{exportId} | GET /v1/identity/accounts/{accountId}/export | Model large or asynchronous exports as jobs/resources. |
| Programme editing | PATCH /v1/programmes/{slug} | POST /v1/programmes/{slug}/update | Use partial updates with optimistic version checking. |
| Programme lifecycle | PATCH /v1/programmes/{slug} with status and retirement reason | POST /v1/programmes/{slug}/archive | Treat archive as a lifecycle state and emit a domain event. |
| Programme category membership | PUT /v1/programmes/{slug}/entity-types/{categoryCode}; DELETE on the same URI | POST .../entity-types/assign and POST .../entity-types/unassign | Model membership as a relationship resource. |
| Content lifecycle | PATCH /v1/programmes/{slug}/content/{contentId} with status/effectiveFrom | POST .../submit, .../review, and .../publish | Keep reviewer identity, notes, version, and effective date in the audit/version record. |
| Policy-draft lifecycle | PATCH /v1/programmes/{slug}/policy-drafts/{draftId} with status/effectiveFrom | POST .../submit, .../review, and .../publish | Use the same lifecycle model as localized content. |
| Geodata imports | POST /v1/geodata/imports using JSON, multipart, or object-reference input; GET/POST `/imports/{runId}/candidates`, `/validate`, `/process`, and `/processed` remain explicit staged subresources | POST /v1/geodata/imports/manual; POST /v1/geodata/imports; POST /v1/geodata/imports/upload; GET `/imports/{runId}` and `/candidates`; POST `/candidates/validate`, `/process`, `/processed` | Consolidate only the three intake representations behind one import collection. Keep preprocessing, validation, promotion, and finalization as explicit resources because they are durable queues and review boundaries. |
| Refresh execution | POST /v1/geodata/refresh-schedules/{scheduleId}/runs; GET /v1/geodata/refresh-runs/{runId} | POST /v1/geodata/refresh-schedules/{scheduleId}/run | Create a refresh-run resource instead of hiding worker execution behind a synchronous action. |
| Manual proposal creation | POST /v1/geodata/proposals | POST /v1/geodata/proposals/draw | Geometry is the representation; the UI may still call the operation draw. |
| Conflation resolution | PATCH /v1/geodata/conflation/{candidateId} or POST .../resolutions | POST /v1/geodata/conflation/{candidateId}/resolve | Use PATCH for one current decision or a resolution subresource for decision history. |
| Entity review decision | POST /v1/geodata/entities/{entityId}/reviews | POST /v1/geodata/entities/{entityId}/review and POST /v1/geodata/entities/{entityId}/status | A review owns the audited lifecycle transition; migrate both current paths to one review resource while retaining the approved→retired invariant. |
| Entity metadata | PATCH /v1/geodata/entities/{entityId} | POST .../name and POST .../location | Use partial updates while preserving manual-field precedence and audit entries. |
| Entity categories | PUT /v1/geodata/entities/{entityId}/categories | POST .../entity-type | Replace the ordered multi-category relationship; first item remains the compatibility primary. |
| Entity geometry | PUT /v1/geodata/entities/{entityId}/geometry | POST .../geometry and POST .../geometry-type | Derive geometry type from GeoJSON, validate allowed kinds, and preserve the edit note. Keep geometry-type conversion as a compatibility operation until all editors submit complete GeoJSON. |
| Entity listing by extent | GET /v1/geodata/entities?bbox=minLon,minLat,maxLon,maxLat | GET /v1/geodata/bbox | Use one entity collection with standard filters and pagination; keep tiles separate. |
| Entity deletion workflow | POST /v1/geodata/entity-deletion-jobs; GET /v1/geodata/entity-deletion-jobs/{jobId} | POST /v1/geodata/entities/{entityId}/delete; GET and POST activation deletion-impact/cascade-delete routes | Keep deletion as a job because it spans QSOs, aggregates, awards, and audit cleanup. |
| Activation close | PATCH /v1/activations/{activationId} with status=CLOSED | POST /v1/activations/{activationId}/close | A close transition is a state update with rule evaluation and audit. |
| QSO batch ingestion | POST /v1/qso-ingestions; GET /v1/qso-ingestions/{ingestionId} | POST /v1/activations/{activationId}/qsos/batch; POST /v1/adif/imports; GET /v1/adif/imports/{importId} | Keep single-QSO POST for interactive entry; represent COPY, ADIF, and high-volume work as one asynchronous ingestion resource with source format and activation scope. |
| QSO corrections | POST /v1/qsos/{qsoId}/corrections; PATCH /v1/qsos/{qsoId}/corrections/{correctionId} | POST /v1/qsos/{qsoId}/corrections; POST /v1/qso-corrections/{correctionId}/review | Nest review under the correction resource rather than using an unrelated global route. |
| Statistics rebuild | POST /v1/statistics/rebuild-jobs; GET .../{jobId} | POST /v1/statistics/rebuild | Treat reproducible aggregation as a job; retain GET /v1/statistics for results. |
| Award lifecycle | PATCH /v1/awards/{awardId} with status/effectiveFrom | POST /v1/awards/{awardId}/submit, /review, /publish, and /retire | Use one versioned award resource; retain audited review decisions. |
| Award recalculation | POST /v1/awards/{awardId}/recalculation-jobs; GET .../{jobId} | POST /v1/awards/{awardId}/recalculate | Create a job because historical definitions affect many participants. |
| Award asset upload | POST /v1/awards/assets/{assetId}/uploads | POST .../upload-url and POST .../content | One upload subresource can return a presigned target or accept multipart fallback. |
| Award evaluation/progress | GET /v1/awards/progress?participantId=...; POST /v1/awards/evaluation-jobs | POST /v1/awards/progress; POST /v1/awards/evaluate | Separate a read query from bulk/recalculation work. |
| Award issuance | POST /v1/awards/requests/{requestId}/issuances; GET /v1/awards/issuances/{issuanceId} | POST /v1/awards/requests/{requestId}/issue | Creating an issuance is resource creation; preserve permanent identity/history. |
| Certificate rendering | POST /v1/awards/issuances/{issuanceId}/render-jobs; GET .../{jobId} | POST /v1/awards/issuances/{issuanceId}/render | Make PDF generation an asynchronous job with retry status. |
| Certificate artifact | GET /v1/awards/issuances/{issuanceId}/artifact | GET /v1/awards/issuances/{issuanceId}/download | Use a resource name supporting content negotiation or signed redirects. |

## Endpoints that should remain explicit

The following are commands or security protocols rather than redundant CRUD
routes: login, refresh, logout, recovery request/reset, service-token
issuance, callsign evidence submission, OIDC provider registration, direct
tile delivery, and worker-triggered compatibility commands.

The rule is not “never use POST to an action URI”. An action URI is justified
when it represents a meaningful domain command or job creation rather than an
ordinary partial update.

## Compatibility and versioning strategy

No existing route should be removed in the first implementation phase.

1. Publish target resources and operation IDs in the canonical OpenAPI file.
2. Add new routes as preferred aliases while old routes emit Deprecation and
   Sunset headers.
3. Make both paths call the same application command/repository method so
   authorization, idempotency, audit, and event behavior cannot diverge.
4. Update generated clients and admin/public web clients to use target paths.
5. Measure old-route traffic and log callers by client/version.
6. Remove aliases only after the documented sunset window and a contract
   compatibility review.

For destructive or cross-service jobs, retain a stable job identifier and
never make a legacy alias silently synchronous. For PATCH operations, require
optimistic version or ETag handling where concurrent admin edits could lose
data.

## Step-by-step implementation plan

### Phase 0 — freeze and reconcile the contract

- [x] Declare [`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml) canonical. See the [Phase 0 contract-freeze record](api-contract-freeze.md).
- [x] Keep root [`myota-contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/openapi.yaml) as a generated mirror, synchronized by [`sync_contract_mirrors.py`](https://github.com/myota-platform/myota-contracts/blob/main/scripts/sync_contract_mirrors.py).
- [x] Make [`myota-platform/contracts/openapi.yaml`](https://github.com/myota-platform/myota-platform/blob/main/contracts/openapi.yaml) a generated/pinned mirror.
- [x] Generate the [`route-inventory.json`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/route-inventory.json) inventory and compare it to every deployed service registry in [CI](https://github.com/myota-platform/myota-contracts/blob/main/.github/workflows/contract-freeze.yml).
- [x] Add checks for duplicate operation IDs, reviewed semantic duplicates, and missing registrations; the reviewed baseline is [`semantic-duplicates.json`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/semantic-duplicates.json).

### Phase 1 — low-risk resource updates

- [ ] Add PATCH programme and account resources.
- [ ] Add idempotent PUT category memberships and primary callsign.
- [ ] Add PATCH award/content/policy-draft lifecycle resources.
- [ ] Keep action routes as aliases and test identical authorization, audit,
  event, and idempotency behavior.

### Phase 2 — geodata resource model

- [ ] Consolidate import representations under POST /geodata/imports.
- [ ] Add POST proposals and PUT entity geometry.
- [ ] Add PATCH entity metadata and PUT entity categories.
- [ ] Add POST entity reviews as the lifecycle decision path.
- [ ] Replace bbox with entity collection filters while retaining tile routes.
- [ ] Introduce the cross-service deletion-job resource with impact and
  confirmation state.

### Phase 3 — activity and award jobs

- [ ] Add activation PATCH close semantics and preserve rule snapshots.
- [ ] Add QSO-ingestion resources for COPY, ADIF, and high-volume submissions.
- [ ] Nest correction review under the QSO correction resource.
- [ ] Add statistics, award recalculation, evaluation, and certificate-render
  job resources with retry and progress status.
- [ ] Add issuance and artifact resources without changing permanent issuance
  semantics.

### Phase 4 — client and operational migration

- [ ] Regenerate the typed client and update admin/public web clients.
- [ ] Update examples, READMEs, diagrams, and operator runbooks.
- [ ] Add dashboards for legacy-route traffic, job lag, failed transitions,
  and alias usage.
- [ ] Test the durable Compose/PostgreSQL stack for contract, authorization,
  idempotency, and audit regressions.

### Phase 5 — deprecation and cleanup

- [ ] Announce target-route availability and sunset dates.
- [ ] Remove aliases only after usage reaches zero or owners explicitly migrate.
- [ ] Remove duplicate contract files or replace them with generated checks.
- [ ] Publish a v1 migration guide and keep this table with the released
  contract.

## Proposed roadmap

- Now: contract reconciliation, route inventory, and target OpenAPI design.
- Next: account, programme, category, and lifecycle resource aliases.
- Then: geodata import/entity/review/deletion jobs.
- After that: activity ingestion, award, statistics, and certificate jobs.
- Release gate: generated clients, web migration, durable-stack tests,
  observability, deprecation headers, and external contract review.

This roadmap improves the transport and resource model used by every
programme; it does not change programme rules.
