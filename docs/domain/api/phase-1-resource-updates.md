# Phase 1 — resource updates

**Status:** complete  
**Completed:** 2026-10-11  
**Owner:** myota-contracts defines the API; Identity and Programme own their handlers.

Phase 1 standardizes ordinary resource changes while keeping audited lifecycle
commands explicit. The canonical operations and schemas are maintained in
[myota-contracts/contracts/openapi.yaml](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).
Deploy and Platform copies are synchronized mirrors.

## Current resource operations

| Bounded context | Operation | Semantics |
| --- | --- | --- |
| Identity account | **PATCH /v1/identity/accounts/{accountId}** | Partial account update; optional roles replaces active role assignments. |
| Identity role | **POST /v1/identity/roles** | Create a custom administrative role. |
| Identity role | **PATCH /v1/identity/roles/{roleCode}** | Update a custom role. |
| Identity assignments | **PUT /v1/identity/accounts/{accountId}/role-assignments** | Replace active role assignments idempotently. |
| Primary callsign | **PUT /v1/identity/accounts/{accountId}/primary-callsign** | Replace the primary callsign idempotently. |
| Programme | **PATCH /v1/programmes/{slug}** | Update programme configuration; archive state is part of resource state. |
| Programme categories | **PUT /v1/programmes/{slug}/entity-types/{categoryCode}** | Ensure the programme/category assignment exists. |
| Programme categories | **DELETE /v1/programmes/{slug}/entity-types/{categoryCode}** | Remove the programme/category assignment. |
| Programme content and policy | Resource reads and updates plus explicit audited lifecycle commands | Draft editing remains separate from submission, review, and publication. |
| Awards | Resource reads and updates plus explicit audited lifecycle commands | Draft editing remains separate from submit, review, publish, and retire decisions. |

The API contract preserves existing scope checks, account and programme
ownership, validation, audit and event writes, and idempotency behavior.
Account deactivation is an account resource update with an explicit status;
it does not bypass the account lifecycle or audit record.

## Evidence

- Contract CI verifies the canonical OpenAPI mirrors and compares the contract
  with current service route registries.
- The route inventory has zero missing contract routes, zero unregistered
  contract operations, and zero duplicate operation identifiers.
- Identity and Programme quality and image workflows pass on current main.
- First-party clients use the canonical resource operations.

See the [REST API consolidation record](rest-consolidation-plan.md), the
[contract freeze](contract-freeze.md), and the
[cross-repository phase evidence](../../architecture/evidence/review-phases-1-4-2026-10-11.md).
