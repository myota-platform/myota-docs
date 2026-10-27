# Phase 6 delegated Participant administration qualification — 2026-10-27

This record covers the deployed `PARTICIPANT_ADMIN` role and Participant
account workspace implemented under
[myota-next issue #97](https://github.com/myota-platform/myota-next/issues/97),
with role scope in [#98](https://github.com/myota-platform/myota-next/issues/98)
and Admin behavior in [#99](https://github.com/myota-platform/myota-next/issues/99).
The work is part of the authenticated Phase 6 workflows in
[issue #91](https://github.com/myota-platform/myota-next/issues/91) and
[issue #10](https://github.com/myota-platform/myota-next/issues/10).

## Deployed release

- Helm release: `myota-test`, chart
  `myota-next-1.1.6-testing.458de92a1f78.1`, app version `testing-33f10fe`,
  revision 43, status `deployed`.
- All 15 application Deployments were ready during the qualification.
- The deployed role catalogue showed the stable code `PARTICIPANT_ADMIN`,
  name `Participant Administrator`, and only the
  `identity.participants.manage` scope.

## Live delegated workflow

- Used an existing synthetic `example.test` Participant account as the support
  identity. A Global Operator temporarily assigned `PARTICIPANT_ADMIN` alongside
  its base `PARTICIPANT` role.
- Signed in to the deployed Admin and used the Participant account workspace to
  search for a separate, active, email-verified synthetic Participant.
- Edited the target's display name. Admin confirmed the update, and the new
  value remained after reloading the browser and searching again.
- Queried the security-event API with the Global Operator account. A
  `identity.account.admin-updated.v1` event matched the exact target account,
  acting support account, and `displayName` field. No account identifiers,
  names, or email addresses are recorded here.
- Direct Participant Administrator requests to the general account list, role
  catalogue, and security-event endpoints each returned HTTP 403. An attempt
  to replace the support account's own role assignments also returned HTTP 403.
- Restored the target's original display name and restored the support account
  to only its base `PARTICIPANT` role after verification. No test account was
  left with elevated authority or a changed display name; the audit events are
  retained as qualification evidence.

## Coverage and remaining qualification

PR [#101](https://github.com/myota-platform/myota-next/pull/101) implemented
the role, scoped Identity APIs, Admin workspace, and service/browser coverage.
Its checks passed before merging to `testing`. The live check above adds the
deployed role-catalogue, display-name persistence, audit attribution, and
out-of-scope denial results.

The Admin browser regression suite still needs to cover paging and display-name
editing/persistence as reusable automated tests. This record also does not
qualify the broader launch-critical Admin workflow or correlated telemetry;
those remain tracked by #91 and #10.
