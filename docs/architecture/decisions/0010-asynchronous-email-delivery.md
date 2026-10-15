# ADR-0010: Asynchronous transactional email delivery

## Status

Accepted on 14 October 2026. Implementation is partial and remains gated.
Identity's internal recipient-resolution API, the versioned email command
contract, and the worker/Helm implementation are merged to monorepo `testing`.
The Testing worker image is published, but the digest-sync workflow skipped
digest resolution because the MyOTA-scoped GitHub App credentials are not
configured. No worker, email stream, or SMTP send path is live. See the [current email record](../../operations/messaging/email-delivery.md)
and [Testing deployment snapshot](../current-state.md).

## Context

Identity owns participant and administrator addresses, eligibility, and
account lifecycle. Sending SMTP from an API request would couple account
operations to provider availability, expose private address data to unrelated
services, and make retries difficult to recover safely. Account-created email
is needed for onboarding, while recovery and verification messages would carry
secret-bearing tokens and require a separate design.

The platform already uses transactional outboxes and bounded JetStream work
queues. Email needs its own work retention, database idempotency, retry and
recovery boundary. SMTP acceptance cannot be committed atomically with the
worker database, so an ambiguous provider acceptance can cause a duplicate.

## Decision

### Ownership and request path

- Identity owns recipient addresses, eligibility, role expansion, account
  lifecycle triggers, and the producer-side durable intent/outbox.
- Contracts owns `email.send.v1`, its immutable schema, bounded template input,
  and source registry.
- `apps/email` owns the worker, template rendering, SMTP interaction, and its
  isolated `myota_email` delivery ledger.
- Deploy owns stream/durable provisioning, secret references, and isolated
  Helm rollout. `myota-docs` owns this decision and the operator record.
- A producer records a stable `workId` and recipient account reference in the
  same database transaction as the selected lifecycle transition. Its relay
  publishes after commit. It never calls SMTP or an email HTTP API in the
  transaction or request path.
- The worker consumes one shared pull durable and resolves the current address
  through Identity's fixed-scope internal API. It must not connect to an
  Identity database. Only the email-service identity may resolve addresses;
  missing, pending-setup, deactivated, and unaddressable accounts fail closed.

### Contract, routing, and privacy

The contract is `email.send.v1` on subject `myota.work.email.send.v1`, stream
`MYOTA_EMAIL_WORK`, durable `email-delivery-v1`. The stream is a separate,
file-backed, single-replica WorkQueue with `DiscardNew`, explicit pull ACKs,
and operator-reviewed finite age/byte/message limits. The numeric email queue
limits have no defaults; Deploy must set them from measured test capacity and
the agreed recovery window before enabling email.

The bounded payload contains a recipient account UUID, an allowlisted versioned
template ID, and template-specific data. The first templates are
`account-created-participant.v1` and
`account-created-administrator.v1`. The payload never contains an email
address, password, access or recovery token, SMTP data, arbitrary HTML, or
message body. Recovery, password reset, and verification mail remain out of
scope until a separate safe one-time token retrieval contract is approved.

### Delivery, state, and recovery

Delivery is **at least once**, not exactly once. The worker commits a bounded
delivery outcome to its own PostgreSQL ledger before ACK, delayed NAK, or
terminal rejection. A crash after SMTP acceptance and before the delivered
state commit can cause a duplicate. JetStream is a bounded recovery window,
not an archive. If a work item expires, recovery must replay its original
stable ID from Identity's producer outbox.

The worker records bounded attempts and redacted dead-letter codes, not raw
commands, resolved addresses, or message bodies. Retries and backoff are finite.
An operator redrive must record a bounded actor, reason, and time, then replay
the same `workId` through the producer outbox. Credentials and message content
must never enter logs, traces, metrics, errors, health responses, or issue
comments. Metric labels are fixed outcomes and allowlisted template IDs.

### Deployment and promotion

The email worker is enabled only in the Helm-backed `myota-test` release after
test-only SMTP, Identity, and database secrets are provisioned, finite queue
limits are measured, and the producer and recipient API are ready. SMTP host,
port, username, sender identity, TLS mode, and timeouts are configuration;
passwords and the Identity client credential are Secret-backed. Test and
production credentials are distinct. Production must never use Testing image
tags or credentials.

Feature PRs do not publish images. A successful push to `testing` publishes
Testing tags, then the unique digest is deployed through the `myota-test`
digest-sync flow. Validate deployment behavior in Helm `myota-test`; do not
use Docker Compose or Podman for deployment tests or qualification. Production
promotion and release-digest checks are outside this decision and require
explicit user direction under repository policy.

## Consequences

- Domain APIs remain independent of SMTP availability and do not receive
  recipient addresses for email delivery.
- Identity, Contracts, Email, Deploy, and Docs have explicit ownership and
  review boundaries.
- The worker database contains only bounded delivery metadata, while the
  producer outbox remains the replay authority.
- Duplicate SMTP delivery remains possible after an ambiguous acceptance and
  must be visible in operator guidance.
- Delivery remains unavailable until producer transactions, queue capacity,
  secrets, real integration checks, image publication, and Helm Testing
  qualification are complete.

## Related work

- [Identity account and role policy](0009-identity-accounts-and-role-policy.md)
- [NATS JetStream event and work topology](0008-nats-jetstream-event-and-work-topology.md)
- [Email delivery operations record](../../operations/messaging/email-delivery.md)
- [Email workstream #58](https://github.com/myota-platform/myota/issues/58)
- Component issues: [Contracts #28](https://github.com/myota-platform/myota/issues/28),
  [Identity #27](https://github.com/myota-platform/myota/issues/27),
  [Deploy #23](https://github.com/myota-platform/myota/issues/23),
  [Docs #25](https://github.com/myota-platform/myota/issues/25),
  [Governance #26](https://github.com/myota-platform/myota/issues/26), and
  [Email #29](https://github.com/myota-platform/myota/issues/29).
