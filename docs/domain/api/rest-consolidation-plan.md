# REST API consolidation — delivered state

**Status:** resource-oriented API delivery, pre-live alias retirement, and
Phase 0 contract-safety closure are complete. See [REVIEW.md](../../../REVIEW.md)
and the [Phase 0 evidence](../../architecture/evidence/review-phase-0-2026-10-11.md).
**Reviewed:** 2026-10-11

## Ownership and contract

[`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml)
is the canonical HTTP contract. Its root-level copy and
[`myota-platform/contracts/openapi.yaml`](https://github.com/myota-platform/myota-platform/blob/main/contracts/openapi.yaml)
are synchronized mirrors. Contracts owns operation IDs, schemas, route inventory,
and dependency-light client facades. Each service owns authorization, validation,
transactions, audit and event writes, idempotency, workers, and persistence.

The deployed service route inventory and canonical contract currently agree:
131 OpenAPI operations and 170 route registrations, with zero unregistered
contract operations, zero runtime routes missing from OpenAPI, and zero duplicate
operation identifiers. Contracts CI rejects duplicate YAML mapping keys with
a Psych AST check and a regression self-test.

## Current API design

The API uses resource operations for ordinary changes and explicit commands for
audited decisions or durable work.

| Area | Current boundary |
| --- | --- |
| Identity | Account, role, callsign, assignment, session, and export resources; authorization and account lifecycle remain Identity-owned. |
| Programme | Programme configuration, category membership, versioned content and policy resources; review and publication remain audited transitions. |
| Geodata | Catalogue, resumable upload sessions, staged imports, review decisions, location enrichment, and deletion jobs; PostGIS and import state remain Geodata-owned. |
| Activity and awards | Activations, QSOs, ingestion/statistics/evaluation/recalculation/render jobs, requests, issuance history, and artifact reads remain Activity-owned. |
| Operations | Storage inspection/history and Grafana identity resolution; NATS metrics are provided by Surveyor and are not an Operations API responsibility. |

Large or retryable work has durable resource state and an owning worker.
Mutations retain their existing service authorization, database transaction,
audit, event, and idempotency boundaries. Audited lifecycle transitions and
job-confirmation boundaries remain explicit commands where they represent a
decision rather than ordinary resource editing.

## Completed implementation

- **Contract baseline:** Contracts owns the canonical OpenAPI, generated route
  inventory, operation-ID checks, and synchronized mirrors.
- **Resource updates:** Identity, Programme, and Awards expose resource-shaped
  updates for ordinary edits while preserving audited lifecycle transitions.
- **Geodata resources:** staged import, resumable upload, entity metadata,
  category, review, enrichment, and deletion workflows use the accepted
  resource/job model.
- **Activity work:** ingestion, rebuild, evaluation, recalculation, and
  certificate rendering expose durable job/resource state.
- **First-party clients:** Admin and participant clients use the current
  contract resources.
- **Pre-live cleanup:** compatibility-only aliases without first-party callers
  were removed from service route tables and the canonical contract. Alias
  response headers, registries, and dedicated metrics were removed as well.
  Audited domain commands and asynchronous job boundaries remain.

No runtime alias-usage instrumentation is present. This cleanup is complete
for the current pre-live clients and contract.

## Verification and follow-up

Contracts CI checks route inventory, mirror consistency, operation identifiers,
and source-to-mirror parity. The 11 October run passed on the current main
revision; see the [cross-repository phase evidence](../../architecture/evidence/review-phases-1-4-2026-10-11.md).
The identity, programme, activity, geodata, operations, deployment, and
integration checks are recorded there with exact commits and workflow runs.

The duplicate YAML mapping-key safeguard is tracked separately in the
[cross-repository architecture review](../../../REVIEW.md). Keep that item open
until a duplicate-key-aware parser check and handler/mirror comparison pass.
