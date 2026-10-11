# API contract and route inventory

**Status:** current contract and service route inventory agree  
**Canonical owner:** myota-contracts  
**Current evidence:** Contract freeze checks, 2026-10-11

## Canonical contract and mirrors

The authoritative OpenAPI document is
[myota-contracts/contracts/openapi.yaml](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).
The repository-local root copy and
[myota-platform/contracts/openapi.yaml](https://github.com/myota-platform/myota-platform/blob/main/contracts/openapi.yaml)
are synchronized mirrors. They are updated from the canonical file and checked
byte-for-byte.

The event registry and schema tree are also Contracts-owned. Platform contains
the synchronized event registry and schema tree. The owner-to-mirror audit
checks exact file and directory contents.

## Route inventory

The checked-in inventory is
[contracts/route-inventory.json](https://github.com/myota-platform/myota-contracts/blob/main/contracts/route-inventory.json).
It scans the runtime route registries in Deploy and the Geodata owner. The
current main-branch inventory contains:

| Check | Current result |
| --- | ---: |
| Canonical OpenAPI operations | 131 |
| Service route registrations | 170 |
| Contract operations without a registered handler | 0 |
| Registered handlers missing from OpenAPI | 0 |
| Duplicate OpenAPI operation identifiers | 0 |

The historical 117-operation baseline from 2026-10-01 is not the current
contract size. Operations were added as resource and job APIs were built, and
unused compatibility-only operations were later removed.

Contracts CI now runs a Psych AST validator before route
inventory generation and self-tests that duplicate keys are rejected without
treating same-named keys in separate mappings as duplicates. The one canonical
Geodata cleanup operation matches its owner handler; the root and Platform
contract mirrors are byte-identical. See the
[Phase 0 closure evidence](../../architecture/evidence/review-phase-0-2026-10-11.md)
and passing [Contracts CI run 38157218999](https://github.com/myota-platform/myota-contracts/actions/runs/38157218999).

## CI gates

[Contract freeze checks](https://github.com/myota-platform/myota-contracts/blob/main/.github/workflows/contract-freeze.yml)
verify canonical contract mirrors, route inventory, operation identifiers,
service owner-to-mirror files, selected schema and migration trees, generated
event schemas, and runtime event-routing catalogs.

Service repositories continue to own request handling, authorization,
idempotency, transactions, audit behavior, event publication, and database
migrations. Contracts does not replace those owners.

See the [REST API consolidation record](rest-consolidation-plan.md) and
[cross-repository phase evidence](../../architecture/evidence/review-phases-1-4-2026-10-11.md).
