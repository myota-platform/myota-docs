# Phase 1 resource updates

Phase 1 of the [REST API consolidation plan](api-rest-consolidation-plan.md)
adds low-risk resource-shaped aliases without removing existing action routes.
The aliases use the same authorization, validation, persistence, audit/event,
and idempotency paths as the existing handlers. This keeps clients compatible
while giving new clients a stable resource-oriented surface.

## Preferred resources

| Bounded context | Preferred route | Replaces or consolidates | Semantics |
| --- | --- | --- | --- |
| Identity | `PATCH /v1/identity/accounts/{accountId}` | `POST /v1/identity/admin/accounts/{accountId}/update` | Partial account update; `roles` may replace active assignments |
| Identity | `POST /v1/identity/roles` | `POST /v1/identity/admin/roles` | Create a custom administrative role as a top-level identity resource |
| Identity | `PATCH /v1/identity/roles/{roleCode}` | `POST /v1/identity/admin/roles/{roleCode}/update` | Partial custom-role update |
| Identity | `PUT /v1/identity/accounts/{accountId}/role-assignments` | `POST /v1/identity/accounts/{accountId}/roles` | Idempotent replacement of active role assignments |
| Identity | `PUT /v1/identity/accounts/{accountId}/primary-callsign` | `POST /v1/identity/accounts/{accountId}/primary-callsign` | Idempotent primary-callsign replacement |
| Programmes | `PATCH /v1/programmes/{slug}` | `POST /v1/programmes/{slug}/update`, `POST /v1/programmes/{slug}/archive` | Partial configuration update; `status: ARCHIVED` retains archive behavior |
| Programmes | `PUT /v1/programmes/{slug}/entity-types/{categoryCode}` | `POST .../entity-types/assign` | Idempotently assign a shared category |
| Programmes | `DELETE /v1/programmes/{slug}/entity-types/{categoryCode}` | `POST .../entity-types/unassign` | Idempotently remove a shared category |
| Programmes | `PATCH /v1/programmes/{slug}/content/{contentId}` | content `submit`, `review`, and `publish` actions | Draft edits and lifecycle transitions selected by `status` |
| Programmes | `PATCH /v1/programmes/{slug}/policy-drafts/{draftId}` | policy-draft `submit`, `review`, and `publish` actions | Draft edits and lifecycle transitions selected by `status` |
| Activity/Awards | `PATCH /v1/awards/{awardId}` | award `submit`, `review`, `publish`, and `retire` actions | Draft edits and lifecycle transitions selected by `status` |

The exact schemas and operation identifiers are maintained in the canonical
[`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).
The root contracts file and the platform copy are generated mirrors; the
mirror check is enforced by
[`contract-freeze.yml`](https://github.com/myota-platform/myota-contracts/blob/main/.github/workflows/contract-freeze.yml).

## Compatibility and safety

The legacy action routes remain registered. They are marked deprecated in the
OpenAPI contract and responses from those routes include `Deprecation: true`
and a `Sunset` header. The default sunset is `2027-04-01T00:00:00Z`; operators
can override it with `MYOTA_LEGACY_ROUTE_SUNSET` during a migration window.

The new aliases delegate to the existing handlers rather than reimplementing
business rules. Consequently they retain the same scope checks, programme and
account ownership checks, audit/event emission, validation errors, and durable
idempotency behavior. `PATCH` lifecycle requests intentionally preserve the
existing state machines: invalid transitions remain errors, and publication
still requires explicit effective dates and publisher identity.

Administrative account deactivation is now sent as a PATCH of the account
resource with `status: DEACTIVATED` and, when requested, `anonymize: true`.
The old `/deactivate` action remains a compatibility route only.

## Verification

Phase 1 was checked with:

```text
python3 -m py_compile common.py identity.py programmes.py
python3 -m py_compile common.py activity.py awards.py
python3 scripts/sync_contract_mirrors.py --platform-root ../myota-platform
python3 scripts/check_contract_phase0.py \
  --canonical contracts/openapi.yaml \
  --mirror openapi.yaml \
  --mirror ../myota-platform/contracts/openapi.yaml \
  --service-root ../myota-deploy/services \
  --semantic-baseline contracts/semantic-duplicates.json \
  --inventory-out contracts/route-inventory.json
```

The contract inventory currently reports zero missing contract routes, zero
missing service registrations, and zero duplicate operation IDs. Service-level
tests cover the existing delegated handlers; the compatibility contract is
also guarded by the route inventory and mirror CI checks.

## Ownership and rollout

Runtime ownership remains split according to the
[repository map](repository-map.md): identity aliases are owned by
`myota-identity-service`, programme aliases by `myota-programme-service`, and
award aliases by `myota-activity-service`. The corresponding copies under
`myota-deploy/services` are the local integration runtime and are synchronized
in the same change. Clients may adopt the preferred routes incrementally;
removal of legacy actions is reserved for a later phase after alias traffic,
authorization, audit, and idempotency telemetry are available.
