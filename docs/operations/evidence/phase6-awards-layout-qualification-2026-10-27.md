# Phase 6 award layout and print qualification

## Scope

This record tracks the published award editor, Award Text, print dimensions, and
background resizing delivered for [issue #104](https://github.com/myota-platform/myota-next/issues/104)
and child issues [#105](https://github.com/myota-platform/myota-next/issues/105),
[#106](https://github.com/myota-platform/myota-next/issues/106),
[#107](https://github.com/myota-platform/myota-next/issues/107), and
[#108](https://github.com/myota-platform/myota-next/issues/108).

## Deployed test release

At qualification time, the Fleet-managed `myota-test` Helm release was healthy:

- Chart: `myota-next-1.1.6-testing.458de92a1f78.1`
- App version: `testing-33f10fe`, Helm revision 43
- Deployments: 16/16 ready
- `/healthz` and `/readyz`: HTTP 200
- Activity image: `sha256:88039880a117dc4d0fba5ba08f8958521ab7e0bc3ec6fee0fc2fb79b009e0fb9`

The deployed award list contained one approved version 1 definition and one
background asset (1491×1055 pixels). A non-mutating preview request returned a
PDF successfully and included the supplied Award Text. Its selected A4 landscape
150 DPI print profile required 1754×1241 pixels. The registered source was below
the minimum 150 DPI detail after cropping, and the API reported
`printReady: false`.

This preview verifies the endpoint and output path only. It does **not** qualify
publication or print readiness for that low-resolution source. No award, issuance,
asset, or PDF record was written by the preview request.

## Code and CI evidence

- [PR #143](https://github.com/myota-platform/myota-next/pull/143) merged to
  `testing` as `5519181f`; its Node CI and image-build/scan checks passed.
- [PR #144](https://github.com/myota-platform/myota-next/pull/144) pins the
  award test images by digest and merged to `testing` as `5ed913ce`.
- [PR #160](https://github.com/myota-platform/myota-next/pull/160) adds a
  published-edit scope-denial service test and Admin browser coverage for
  validation and stale-version conflict feedback. It merged as `1d54a05` after
  Node CI and all image-build/scan checks passed.
- Post-merge Node CI and the testing image-source workflow passed. No application
  image changed: the Helm release remained at chart `myota-next-1.1.6-testing.458de92a1f78.1`,
  app `testing-33f10fe`, revision 43. All 16 deployments remained ready, and
  `/healthz` and `/readyz` returned HTTP 200.
- The Activity compatibility tests cover published-version creation, stale
  version conflicts, Award Text validation, backwards-compatible empty text,
  supported print profiles, and source-resolution readiness.
- Renderer tests cover all A4/Letter portrait/landscape dimensions at 300 DPI,
  resizing a mismatched background without modifying its source, and rendering
  Award Text.
- The Admin browser test covers loading a published version, creating a draft
  revision, default Award Text placement, A4 landscape dimensions, and saving
  with the expected source version. Its API is fixture-backed.
- The CI live awards browser workflow uses a mismatched high-resolution
  background and exercises publish, request, issue, and PDF rendering through
  the Activity service and worker.

## Remaining evidence gaps

- Add an HTTP-level authorization-denial test for the published-edit endpoint;
  PR #160 currently proves scope denial at the Activity service boundary.
- Verify a saved award revision and print-ready publication against the deployed
  test service using a sufficiently detailed background.
- The currently deployed low-resolution asset cannot verify successful
  publication. A suitable high-resolution source is needed for a live
  `myota-test` publication and print-readiness check.
- The Admin browser editor test uses API fixtures; it does not exercise the
  deployed browser against the current `myota-test` API.

Do not mark the four child issues or issue #104 complete until their remaining
acceptance criteria are verified and linked from the issues and Phase 6
qualification issue #91.
