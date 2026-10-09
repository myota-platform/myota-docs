# API contract freeze — Phase 0

Status: complete  
Reviewed: 2026-10-01  
Owner: `myota-contracts` with the deployed service registries

Phase 0 establishes the contract baseline used before any REST route
consolidation. It does not remove or change runtime routes.

## Canonical contract and mirrors

The canonical OpenAPI document is:

- [`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml)

The following files are retained as generated mirrors for consumers that
already depend on their repository-local path:

- [`myota-contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/openapi.yaml)
- [`myota-platform/contracts/openapi.yaml`](https://github.com/myota-platform/myota-platform/blob/main/contracts/openapi.yaml)

They must not be edited independently. Run
`scripts/sync_contract_mirrors.py --platform-root ../myota-platform` from the
contracts checkout after changing the canonical document. CI fails if either
mirror differs byte-for-byte from the canonical file.

## Route reconciliation

The generated inventory is [`route-inventory.json`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/route-inventory.json).
The Phase 0 checker scans the five HTTP registries used by the deployed local
runtime in `myota-deploy/services`: identity, programme, geodata, activity,
and awards. It compares method/path pairs with the canonical OpenAPI paths.

Baseline recorded on 2026-10-01:

| Check | Result |
| --- | ---: |
| Canonical OpenAPI operations | 117 |
| Deployed registry routes | 117 |
| Registered routes missing from contract | 0 |
| Contract operations missing from registry | 0 |
| Duplicate OpenAPI operation IDs | 0 |
| Reviewed semantic-duplicate groups | 7 |

The checker is [`check_contract_phase0.py`](https://github.com/myota-platform/myota-contracts/blob/main/scripts/check_contract_phase0.py),
and the review baseline for intentional action-style overlaps is
[`semantic-duplicates.json`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/semantic-duplicates.json).
The semantic groups are not treated as accidental duplicates: they represent
audited lifecycle, relationship, deletion, verification, publication, or
recalculation commands that remain explicit until a later migration phase.
Any new or changed group fails CI until it is reviewed and added to that
baseline with a rationale.

## CI gate

The executable gate is the
[Contract freeze checks workflow](https://github.com/myota-platform/myota-contracts/blob/main/.github/workflows/contract-freeze.yml).
It validates YAML syntax, verifies both mirrors, compares the route inventory,
checks duplicate operation IDs and semantic-duplicate drift, and ensures the
generated inventory has no uncommitted changes.

Phase 1 may now add preferred resource aliases, but it must preserve the
frozen routes, authorization, idempotency, audit, and event behavior until
client migration and the documented deprecation window are complete.
