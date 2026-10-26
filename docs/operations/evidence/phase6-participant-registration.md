# Phase 6 Participant registration qualification evidence — 2026-10-26

This record covers the real Participant registration and email-verification
journey against the isolated `myota-test` deployment. It uses a synthetic
`example.test` address and the deployment's Mailpit SMTP sink; no real mailbox
or production account was used.

Related work: [Participant workflow #96][issue-96], [registration #100][issue-100],
[authenticated workflow #91][issue-91], and [Phase 6 #10][issue-10].

[issue-96]: https://github.com/myota-platform/myota-next/issues/96
[issue-100]: https://github.com/myota-platform/myota-next/issues/100
[issue-91]: https://github.com/myota-platform/myota-next/issues/91
[issue-10]: https://github.com/myota-platform/myota-next/issues/10

## Deployed revision

- Helm release: `myota-test`, chart `myota-next-1.1.6-testing.492a44b.1`,
  app version `testing-492a44b`, revision 26.
- Identity image: `ghcr.io/myota-platform/myota-next-identity`, digest
  `sha256:847976766742f709203bc92ffdc29f21e92b021f5159dadb25af1840fe1add48`.
- Participant web image: `ghcr.io/myota-platform/myota-next-participant-web`, digest
  `sha256:87f3d162473b4a2efac7f17a99b37920e1b7ae1640977a221b89d775b9dae73c`.
- Participant page: `https://myota-participant-test.spainip.es/`.
  The API was reached through the configured same-origin `/v1/` route.

The tested application images are the ones currently deployed in revision 26;
this is not a claim that the newer backup tooling from PR #119 has been deployed.

## Procedure and results

1. Opened the deployed Participant page in Chromium and submitted a synthetic
   registration with display name, `example.test` email, password, and Operator
   participant type.
2. Confirmed that the page returned the generic “If registration can proceed”
   response and that the message arrived in the Mailpit sink.
3. Followed the verification link in the captured message. Identity activated
   the account, the Participant page reported success, and the token was removed
   from the browser URL.
4. Signed in with the new credentials. The live `/v1/identity/me` response was
   `ACTIVE` with only the `PARTICIPANT` role. Signed out and confirmed the page
   returned to the signed-out state.
5. Deleted the synthetic verification email from Mailpit after use. The
   synthetic participant account remains in the isolated database for continued
   Participant workflow qualification; its generated password is not recorded
   in this repository or issue comments.

The post-merge Node CI run for PR #119 passed, including the Identity and browser
suites. Service tests cover participant-only role assignment, token replay
rejection, generic responses for an existing unverified address, and denial of
login before verification.

## Remaining work

This is not sufficient to close #100 or #91. Add explicit automated coverage and
deployment checks for expired and malformed tokens, registration and resend
throttling under abuse, resend behavior for verified and unverified accounts,
and duplicate-address outcomes. Continue with the profile, callsign, activation,
QSO, and ADIF journeys tracked by #96. Link those results here and from the
Phase 6 readiness issue before changing its exit criteria.
