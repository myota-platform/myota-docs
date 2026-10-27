# Phase 6 Participant registration qualification evidence

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

## Initial registration qualification revision

- Helm release: `myota-test`, chart `myota-next-1.1.6-testing.492a44b.1`,
  app version `testing-492a44b`, revision 26.
- Identity image: `ghcr.io/myota-platform/myota-next-identity`, digest
  `sha256:847976766742f709203bc92ffdc29f21e92b021f5159dadb25af1840fe1add48`.
- Participant web image: `ghcr.io/myota-platform/myota-next-participant-web`, digest
  `sha256:87f3d162473b4a2efac7f17a99b37920e1b7ae1640977a221b89d775b9dae73c`.
- Participant page: `https://myota-participant-test.spainip.es/`.
  The API was reached through the configured same-origin `/v1/` route.

This records the release used for the initial registration happy path. The
current `myota-test` Helm release is revision 32, chart
`myota-next-1.1.6-testing.c10df2f22479.1`; the live throttle recheck below was
performed against the current public Participant endpoint.

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

## Abuse controls and invalid-link regressions — 2026-10-27

PR #136 merged to `testing` as commit
[`1083718`](https://github.com/myota-platform/myota-next/commit/1083718a362c1bec66bb9386d95c245ae53b0b0d).
The merge request's Node CI and all image build/scan checks passed; the
post-merge Node CI [run](https://github.com/myota-platform/myota-next/actions/runs/40427700456)
and testing publisher [run](https://github.com/myota-platform/myota-next/actions/runs/40427700526)
also succeeded. The change adds Identity tests for malformed and expired
tokens, resend token replacement, generic responses for verified addresses,
and the five-request throttle, plus a Participant browser check that an
invalid verification token is removed from the URL. The focused Identity
suite passed 7 tests and the Participant registration browser suite passed 2.

On the live endpoint, five resends to a unique unregistered `example.test`
address returned the same generic HTTP 200 response; the sixth returned HTTP
400 with problem code `invalid_request` and a rate-limit detail. A malformed
verification token also returned HTTP 400. These requests did not create
accounts; the hashed rate-limit records expire after 15 minutes.

## Follow-up: profile and callsigns

The same synthetic participant signed in and saved a profile website and short
biography. Both values remained after a browser reload. The participant added a
synthetic callsign, which appeared as `UNVERIFIED` and `Primary`, and it also
remained after reload. The participant then signed out. The synthetic callsign
is test-only and has no verification evidence.

## Follow-up: activation, manual QSO, and closure — 2026-10-27

This follow-up supersedes the earlier note that the public catalogue had no
selectable entity. The current test deployment includes a synthetic Phase 6 QA
entity. A signed-in Participant account with only the `PARTICIPANT` role had
an open activation with one QSO from the previously qualified ADIF import. In
the deployed Participant browser, the account recorded another contact
(Activity API HTTP 201), closed the activation (HTTP 200), signed out, and
signed back in. The activation remained `CLOSED` with two QSOs in the page and
the Activity API response. The account, entity, activation, and contact are
synthetic test data.

This verifies the browser-to-API path for manual QSO entry, activation closure,
and persistence across a fresh session. That activation is now closed. QSO
correction/edit behavior and correlated logs, metrics, traces, and audit
evidence remain open under #91.

## Fresh-account profile, callsign, activation, QSO, and ADIF qualification — 2026-10-27

This follow-up used a fresh synthetic Participant because an older local test
credential no longer authenticated. The fresh account was registered and
email-verified through the deployed Participant web and the test Mailpit SMTP
sink. Its generated credential remains in the ignored local `runtime-secrets/`
directory and is not included here.

The isolated release was Helm `myota-test`, chart
`myota-next-1.1.6-testing.458de92a1f78.1`, app version `testing-33f10fe`,
revision 43. All 15 application Deployments were ready at the check; the
release's `/healthz` and `/readyz`, Participant web, and Admin web returned
HTTP 200. The relevant immutable images were:

- Identity: `sha256:e9904e04ca1227af302be100e7090500be4b5ef8499aeadcf84e5ad78a8b04b0`.
- Activity API: `sha256:88039880a117dc4d0fba5ba08f8958521ab7e0bc3ec6fee0fc2fb79b009e0fb9`.
- Activity worker: `sha256:74cd1cb279a078451e0172690f6b434c72bbd0b4795abf2245fb50f37060d151`.
- Participant web: `sha256:a324b3d01fa9edbe7463215e55d7eb11f1eea3bc64394fc38cca070765755623`.

The browser ran at 390×844. The Participant added a synthetic callsign and
submitted evidence explicitly marked as a test fixture. An Admin reviewed and
verified that callsign through the Admin identity UI so the Participant could
exercise programme rules that require a verified callsign. This synthetic
fixture does not represent verification of a real operator's callsign.

The Participant then completed these deployed journeys:

- Signed in with Enter, edited the synthetic profile, saved it with Enter, and
  confirmed the website and biography remained after reloading the browser.
- Started and closed an activation after recording two manual QSOs. The API
  returned `CLOSED`, `ruleEvaluation.valid=true`, and a count of two.
- Started another activation, uploaded a one-record ADIF file, and observed the
  terminal import result `COMPLETED`, one accepted, and zero rejected. After a
  manual QSO, this activation also closed with `ruleEvaluation.valid=true` and
  two QSOs, one from ADIF and one manual.
- Re-read both activation records and the ADIF import through the authenticated
  Activity API. Both records persisted with no rule violations, and the import
  returned HTTP 200 with no errors.

The generated verification email was deleted from Mailpit after use. The
synthetic Participant, callsign, evidence, activations, and QSOs remain in the
isolated test database for further qualification. The earlier unrelated failed
ADIF import remains test data and is not counted as a passing result.

This closes the deployed minimum Participant self-service journey for #96,
including mobile-sized and keyboard use. Cross-participant detail isolation is
also covered by [activation-history evidence][activation-history]. Observe
telemetry delivery, complete Phase 6 observability, and the broader Admin
readiness journey remain open under #91 and #10.

[activation-history]: https://github.com/myota-platform/myota-docs/blob/main/docs/operations/evidence/phase6-participant-activation-history-2026-10-27.md
