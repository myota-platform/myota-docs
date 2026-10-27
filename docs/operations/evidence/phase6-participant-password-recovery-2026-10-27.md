# Phase 6 Participant password recovery qualification — 2026-10-27

This qualification covers the deployed password-reset flow tracked by
[myota-next issue #114](https://github.com/myota-platform/myota-next/issues/114),
under Phase 6 issue [#96](https://github.com/myota-platform/myota-next/issues/96)
and overall qualification [#10](https://github.com/myota-platform/myota-next/issues/10).

## Deployment

- Helm release: `myota-test`, chart
  `myota-next-1.1.6-testing.458de92a1f78.1`, app version `testing-33f10fe`,
  revision 43, status `deployed`.
- All 15 application Deployments were ready during the qualification.
- Password-reset email was delivered to the existing synthetic Participant
  through the isolated in-cluster Mailpit SMTP sink.

## Live flow and outcomes

- Submitted the reset request through the deployed Participant web. The UI
  displayed the generic “If an account matches that address” response, and the
  configured email sink received the reset message.
- With the synthetic account's existing access and refresh credentials held
  only in memory, completed reset requests and password changes through the
  deployed Identity API. All four operations returned HTTP 200.
- The old access credential was denied on the authenticated account endpoint
  with HTTP 403; the old refresh credential was also denied with HTTP 403.
- Signed in successfully with the temporary new password, then reset the
  account back to its original password and verified that sign-in succeeded.
- Removed all five reset emails generated across the live reset and Participant
  UI checks. The two older verification messages already in Mailpit were
  preserved. No password, token, message body, or account identifier is
  recorded in this document.

## Limits and remaining work

This verifies deployed email delivery, the successful reset path, and rejection
of the pre-reset access and refresh credentials. It does not qualify abuse
throttling under load, reset-token expiry or replay against the deployed
service, or production SMTP delivery and trust roots. The existing automated
service and browser coverage remains the evidence for those code paths. Keep
#114 open until its remaining task and exit criteria are verified, including
the explicit telemetry and security requirements.
