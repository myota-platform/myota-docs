# Identity accounts and role-grant policy

**Status:** Decision for implementation; runtime rollout is pending.
**Date:** 2026-10-14
**Owner:** Identity (myota/apps/identity); Deploy owns the synchronized
Compose runtime copy until that deployment path is retired.

## Decision

MyOTA uses one Identity account for each person. Participant and administrator
access are represented by assigned Identity roles and permissions, not
separate account populations or client-supplied account types.

The built-in PARTICIPANT role has no administrative permissions. Identity
adds the baseline identity.me scope to authenticated accounts. Creating an
account requires identity.accounts.create. Additional roles require the
separate identity.roles.assign capability. Identity allows a role grant only
when its permission set is a strict subset of the actor's current permission
set; role mutations cannot target the actor. A role with equal or greater
authority requires an offline or deployment-controlled bootstrap decision.

New accounts enter PENDING_SETUP; account-creation requests cannot contain a
permanent password. Invitation and credential setup must use the asynchronous
email flow once deployed. Public participant registration remains unavailable
until email verification, abuse prevention, and deployment gates are complete.
Registration responses must not reveal whether an address already exists.

## Security and compatibility constraints

- Identity is the authority for accounts, role assignments, and permission
  decisions. Gateway and browser code may present permissions but cannot
  grant them.
- Account creation uses the existing in-process idempotency guard when an
  Idempotency-Key is supplied and rejects duplicate normalized email addresses.
  Database-level, cross-replica uniqueness and idempotency await relational auth
  persistence.
- Access-token authorization resolves current roles on each request within the
  handling process, so role removal and account deactivation take effect before
  the token expires there. Cross-replica consistency remains unqualified.
- Signing keys stay in deployment secrets. Password hashes, invitation tokens,
  and SMTP credentials never enter API responses, logs, or NATS payloads.
- The relational PostgreSQL auth schema must become the runtime authority
  before multi-replica Identity deployment is considered qualified.

## Current implementation evidence

The account-creation and grant-authority implementation merged to monorepo
`testing` in PR [#70](https://github.com/myota-platform/myota/pull/70), followed
by effective-permission refresh and delegated-role enforcement in PR
[#72](https://github.com/myota-platform/myota/pull/72). Their reported PR
checks passed. The source implements one account record with a baseline
`PARTICIPANT` role, requires `identity.accounts.create` for account creation,
requires `identity.roles.assign` for additional roles, and rejects self- or
equal/higher-authority grants. New accounts remain `PENDING_SETUP`; permanent
passwords are rejected, and the public registration route remains disabled.
These are merged-source facts; this paragraph does not claim a current Helm
runtime qualification.

The current Identity process still hydrates accounts, credentials, role
assignments, sessions, recovery state, revocations, OIDC mappings, and security
events into process memory from the PostgreSQL `service_state` JSONB snapshot
in `apps/identity/common.py`. The Core baseline already defines relational
account, credential, role, session, recovery, revocation, callsign, OIDC, and
security-event tables, but Identity does not yet use them as its runtime source
of truth. Cross-replica uniqueness and refresh/revocation/lockout/recovery
consistency, plus backup and restore after a relational cutover, remain open
under
[issue #36](https://github.com/myota-platform/myota/issues/36) and
[Deploy issue #35](https://github.com/myota-platform/myota/issues/35).

The authentication state inventory and proposed relational mapping are in
[Identity security and persistence](../../domain/identity/security.md).

## Follow-up work

- [ ] Complete relational Identity persistence and restartable state migration.
- [ ] Implement invitation and credential setup through the email service.
- [ ] Complete Admin Web role management and secure account creation.
- [ ] Verify Gateway authorization/idempotency forwarding and deployment order.
- [ ] Keep public registration disabled until identity verification and
      anti-abuse gates are qualified.
- [ ] Update this record with merged/deployed evidence before closing the
      Identity workstream.

See [workstream #60](https://github.com/myota-platform/myota/issues/60),
[Identity #19](https://github.com/myota-platform/myota/issues/19), and
[secure account creation #34](https://github.com/myota-platform/myota/issues/34),
[Admin UI role management #14](https://github.com/myota-platform/myota/issues/14),
[shared account contracts #15](https://github.com/myota-platform/myota/issues/15),
[participant registration #16](https://github.com/myota-platform/myota/issues/16),
[registration deployment #17](https://github.com/myota-platform/myota/issues/17),
[shared Identity documentation #18](https://github.com/myota-platform/myota/issues/18),
and [Identity schema deployment #35](https://github.com/myota-platform/myota/issues/35).
