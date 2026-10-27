# Phase 6 Participant activation-history qualification — 2026-10-27

This record covers the Participant activation detail view implemented for
[myota-next issue #154](https://github.com/myota-platform/myota-next/issues/154),
under the Participant self-service work in
[issue #96](https://github.com/myota-platform/myota-next/issues/96). The change
was delivered by [PR #155](https://github.com/myota-platform/myota-next/pull/155)
and the isolated release image digests were updated through
[PR #156](https://github.com/myota-platform/myota-next/pull/156).

## Deployed release

- Helm release: `myota-test`, chart
  `myota-next-1.1.6-testing.458de92a1f78.1`, app version `testing-33f10fe`,
  revision 43, status `deployed`.
- All 15 application Deployments had their requested replicas ready at the
  check. The release uses immutable image digests.
- `https://myota-test.spainip.es/healthz` and `/readyz` both returned HTTP 200.

## Workflow and authorization results

- Registered and email-verified a fresh test Participant through the
  Participant web using the deployment's Mailpit SMTP sink.
- Created and closed a test activation, recorded a manual QSO, and uploaded a
  small ADIF file. The Activity worker completed the import with one accepted
  record and no rejected records.
- The activation owner read the activation detail through the Activity API and
  saw the stored activation, both QSOs, and the completed ADIF import outcome.
  The response contained only the safe import summary; it did not expose
  storage bucket/object keys, content hashes, or internal error details.
- A second verified Participant received HTTP 404 with the same generic
  `Activation not found` response used for a nonexistent activation. This
  prevents the detail endpoint from disclosing another Participant's record.
- In the Participant web, the owner opened the accessible activation detail
  disclosure and saw both QSO records and the import result. Browser coverage
  also checks keyboard activation, safe rendering of user-controlled filenames,
  and responsive layouts at 390 px and 1280 px.
- Activity service tests cover owner-only access, the indistinguishable
  not-found response for non-owners, administrator access, and repository
  mapping of QSO import links and safe import fields. The Participant browser
  regression test verifies the rendered detail and avoids loading it before
  the disclosure is opened.

## Limits and remaining qualification

This evidence qualifies the activation-history slice of #96 and the deployed
Activity API and Participant UI path. It does not qualify the wider Participant
profile/callsign workflows, delegated Participant administration, Observe
telemetry, backup and restore, or the full launch-critical Admin journey.
Those remain tracked under [Phase 6 issue #10](https://github.com/myota-platform/myota-next/issues/10)
and [authenticated workflow issue #91](https://github.com/myota-platform/myota-next/issues/91).

Test accounts and the test activation/import were created in the isolated
`myota-test` environment. Production data and deployments were not used.
