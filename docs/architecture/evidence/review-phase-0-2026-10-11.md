# Cross-repository review — Phase 0 contract safety

**Completed:** 2026-10-11  
**Team:** Codex and Volker  
**Scope:** myota-contracts canonical OpenAPI; the Contracts root mirror; the
myota-platform OpenAPI mirror; Geodata handler evidence; and myota-docs status.

## Prompt used

~~~text
Decide how to resolve: Phase 0 remains open for duplicate-YAML-key rejection
and the final handler/mirror check.
~~~

## Decision

Use a standard-library Psych AST check in Contracts CI. The check walks every
mapping below YAML stream and document nodes, rejects repeated scalar keys in
the same mapping, and rejects unsupported non-scalar keys. A CI self-test
proves that a duplicate key fails and that identical names in separate
mappings remain valid. Keep the existing YAML syntax and route-inventory
checks as separate gates.

The two Geodata cleanup path entries were one duplicate mapping key, not two
operations. The canonical contract now has one `deleteGeodataLoadTestRun`
operation, with the handler-derived `lt-` identifier pattern and one complete
response schema. The response requires and returns `testRunId`,
`importsDeleted`, `entitiesDeleted`, `objectsDeleted`,
`uploadSessionsDeleted`, and `cleaned: true`.

## Handler and safety comparison

The Geodata owner is
[geodata.py — `GeoHandler.cleanup_load_test_run`](https://github.com/myota-platform/myota-geodata-service/blob/main/geodata.py).
Its destructive-operation protections are represented in the contract:

- Explicit `MYOTA_LOAD_TEST_CLEANUP_ENABLED=1` and a declared environment;
  production also requires `MYOTA_LOAD_TEST_ALLOW_PRODUCTION_CLEANUP=YES`.
- Bearer authentication and the supported `GLOBAL_ADMIN` or
  `GLOBAL_OPERATOR` role.
- Exact `DELETE LOAD TEST DATA <testRunId>` confirmation and the handler's
  `^lt-[A-Za-z0-9._-]{1,77}$` identifier constraint.
- Refusal while import runs or promotion queues are active; refusal to delete
  objects outside the Geodata import bucket or upload spool.
- Cross-service activity checks protect QSOs, activations, and award progress.
- The handler's response fields and types are fully listed in OpenAPI.

The focused handler safety cases are maintained in
[tests/test_load_test_cleanup_safety.py](https://github.com/myota-platform/myota-geodata-service/blob/main/tests/test_load_test_cleanup_safety.py).
No Geodata runtime file was changed in this phase.

## Changes

### myota-contracts

- `contracts/openapi.yaml`: collapse duplicate path declarations, retain the
  operation ID and strict fixture identifier pattern, reconcile authorization
  and deletion safeguards, and describe the full response.
- `openapi.yaml`: byte-identical repository-root mirror.
- `contracts/route-inventory.json`: regenerate from the canonical source.
- `scripts/check_yaml_duplicate_keys.rb`: add duplicate-key validation with
  a self-test for duplicate rejection and mapping-scope correctness.
- `.github/workflows/contract-freeze.yml`: run the self-test and validator
  before YAML syntax and route-inventory verification.

Canonical contract hash: `f44f1c6cfd7abe9c838120ffb4b44c32e780c73242121df66a7e9a76cf060206`.
The regenerated inventory has 131 unique OpenAPI operations, 170 service
route registrations, and zero missing routes, duplicate operations, or
duplicate operation IDs. The prior count of 132 counted the duplicate
cleanup declaration twice.

### myota-platform

- `contracts/openapi.yaml`: synchronized byte-for-byte from Contracts.
  Platform remains an integration/bootstrap mirror, not an independent owner.

### myota-docs

Updated `REVIEW.md`, the API contract freeze and REST consolidation pages,
To do, Work in progress, prioritized backlog, changes, and implementation
timeline. This evidence file is the source for the Phase 0 completion links.

## Verification

- Contracts commit `1ed6efb476879da3d10abb20325a104e5f504e24` introduced the
  contract correction and validator. The first CI run exposed a traversal
  defect in the new checker self-test; it correctly prevented the item from
  passing.
- Follow-up Contracts commit
  `36cf244bedaf6071e50d7c5aef973ccd890128c4` fixed traversal through Psych's
  stream/document nodes.
- [Contracts CI run 38157218999](https://github.com/myota-platform/myota-contracts/actions/runs/38157218999)
  passed. The duplicate-key self-test, canonical OpenAPI rejection check,
  YAML syntax check, frozen route inventory, mirror audit, and committed
  inventory check all passed.
- [Platform CI run 38157179490](https://github.com/myota-platform/myota-platform/actions/runs/38157179490)
  passed on mirror commit
  [2426e4da186aca49cfa00a58a83746f3573cdf66](https://github.com/myota-platform/myota-platform/commit/2426e4da186aca49cfa00a58a83746f3573cdf66).
- A direct read-back confirmed that Contracts' canonical file, its root copy,
  and Platform's copy are byte-identical. The cleanup path occurs once in the
  OpenAPI document.
- No runtime code, database schema, deployment, or live data was changed.

**Phase 0 exit criteria:** complete. The duplicate declaration is removed,
the YAML validator and self-test pass, the handler comparison is recorded,
all OpenAPI mirrors match, and the route inventory is current.
