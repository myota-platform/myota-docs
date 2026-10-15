# Identity, authentication and authorization

The identity service owns accounts, callsigns, credentials, sessions, roles,
scope grants, OIDC programme mappings, recovery tokens and security events. It
does not depend on Keycloak.

## Authentication and account lifecycle

Identity has one account record per person. `participationType` (`OPERATOR` or
`SWL`) describes the radio participation profile; it does not grant
administrator access. Each new account created through the authorized account
API receives the least-privilege `PARTICIPANT` role and enters `PENDING_SETUP`.
The API rejects
permanent passwords during account creation. Future public registration is
disabled until verified-email, non-enumeration, rate-limit, and deployment
gates are complete; clients cannot choose roles or permissions.

Existing credentials use salted scrypt password hashes. Login returns a
ten-minute HMAC-SHA-256 access token and a refresh token with a default
30-day lifetime. Refresh is
rotating: the old session is revoked before a new refresh token is issued.
Logout, password recovery, deactivation, and token revocation invalidate active
sessions. Recovery requests return the same public response whether or not the
email exists. Development reset tokens are not exposed in production. Signing
keys are supplied through `MYOTA_AUTH_SIGNING_KEY`; bootstrap administrator
email/password, when configured, are deployment-provided secrets and are never
stored as plaintext state. Production refuses the built-in development signing
key. Services that validate a token share the key through a Kubernetes Secret,
while service-to-service tokens are short-lived, scope-limited tokens issued
through the Identity API.

## Callsigns and authorization

An account can have many callsigns but at most one active primary callsign.
Evidence records preserve type, source, checksum and metadata. A callsign cannot
be retired while primary without an active replacement. Verification requires
pending evidence and the `callsign.verify` scope.

Roles can be global or scoped to a programme, jurisdiction and entity type.
The built-in administration hierarchy includes Global administrator, Identity
administrator, GIS administrator, Geodata approver, Programme administrator,
Activity administrator and Auditor. Global administrators can create custom
roles from the allowlisted administrative permission catalogue; wildcard
access remains reserved for the global administrator. The geodata service
checks the signed role claims before accepting a review; a `GEO_APPROVER`
grant for one programme or entity type cannot review another. Roles and scope
changes produce audit/security events.

Identity is the authorization authority. Account creation requires
`identity.accounts.create`; adding administrator roles also requires
`identity.roles.assign`. Identity rejects self-assignment, roles with equal or
higher permissions than the actor's, and removal of the last global
administrator. Admin UI controls are presentation only; Identity enforces every
grant. Effective permissions are recalculated from the role assignments in
Identity's process-local state when an access token is authorized. Immediate
cross-replica consistency awaits the relational cutover described below.

Per-programme OIDC provider mappings store issuer, client ID, requested scopes
and enabled state. These mappings define the internal-account integration
boundary; provider secrets and authorization-code exchange belong in a future
provider adapter and deployment secret store, never in public programme config.

## Privacy and abuse controls

Accounts can export their account, roles, evidence and security events, or
deactivate with anonymization. The general login-attempt throttle is keyed by
email and source address; accounts assigned the built-in `GLOBAL_OPERATOR`
role bypass that throttle to avoid locking out platform administrators during
repeated legitimate sign-ins. The separate temporary account lock after five
invalid passwords still applies to global operators. Other accounts remain
subject to both controls. Login/recovery/role/session events are auditable
through the durable outbox. Retention jobs should purge expired recovery
tokens, revoked sessions and security events according to the deployment's
policy.

## Authentication-state inventory and persistence boundary

The current runtime still uses `Store` in `apps/identity/common.py`. It loads
`service_state.state` for service `identity` into process memory and writes the
aggregate back after requests. The database is durable storage, but the JSONB
aggregate and process-local cache are still the runtime model; they do not
provide safe cross-replica read/modify/write behavior. The Core baseline's
relational tables are not yet Identity's runtime source of truth. This is the
remaining scope of [Identity issue #36](https://github.com/myota-platform/myota/issues/36)
and [Deploy issue #35](https://github.com/myota-platform/myota/issues/35).

| Current JSON state | Data and security purpose | Relational target already in Core baseline | Cutover gap |
|---|---|---|---|
| `items` | Account ID, normalized email, status, participation type, profile metadata, callsigns, and primary callsign | `account`, an Identity-owned profile extension, `callsign`, `account_primary_callsign` | Case-insensitive unique email and complete profile/callsign migration are not enforced by Identity's runtime. |
| `data.credentials` | Password hash, password-change time, failed attempts, and temporary lockout | `account_credential` | Login and password changes currently read/write the in-memory JSON projection. |
| `data.roles`, `data.roleDefinitions` | Assigned roles, scopes, validity, and built-in/custom role definitions | `account_role`; role-definition persistence needs an Identity-owned table/model | Grant checks and effective permissions use process memory. |
| `data.sessions` | Refresh-token hash, access-token JTI, expiry, revocation, and bounded client metadata | `auth_session` | Rotation/revocation is not serialized across replicas. |
| `data.revokedJti` | Access-token revocation JTI and expiry | `auth_revocation` | Token checks use process memory. |
| `data.recovery` | SHA-256 token hash, account, expiry, and use time; raw token is returned only under the development gate | `account_recovery_token` | Single-use consumption and password reset are not transactionally enforced across replicas. |
| `data.rateLimits` | Expiring login attempt key (email plus remote address), window, and count | No matching Core table | A shared, expiry-aware rate-limit store is needed; this key is sensitive and raw email must not become a metric label. |
| `data.securityEvents` and `events` | Security audit history and transactional outbox records | `security_event`, `outbox_event` | Audit/outbox creation must commit atomically with the account or auth transition. |
| `data.oidcMappings` | Issuer/client/scope configuration for programme OIDC mapping | `oidc_provider_mapping` | Runtime currently reads the JSON projection. Provider secrets remain outside database/config records. |
| `idempotency` | Cached API result by idempotency key | `idempotency_record` | Account creation idempotency is currently process-local before persistence. |
| Bootstrap environment | Optional initial administrator email/password | Kubernetes Secret / runtime environment; resulting account, credential hash, role, and audit row belong in the relational targets above | Startup currently checks and mutates the process-local projection; it must be idempotent against database constraints. |

The migration must preserve every existing account ID, password hash, active
role, session/revocation, recovery record, callsign, and audit event. It must
run restartably with pre/post counts and deterministic checksums, fail closed
on ambiguous duplicates or malformed legacy data, and retain a tested rollback
and backup/restore procedure. No plaintext password or signing key belongs in
the schema. The intended migration and runtime cutover are not yet implemented.
