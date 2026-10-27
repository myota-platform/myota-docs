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

After PR #160, a separate synthetic qualification award was created through the
deployed Activity API. Its registered A4 landscape background source measured
3167×2240 pixels. The saved award was published as version 1 with Award Text,
and a subsequent read confirmed that the text and layout element persisted.
The source passed the 150 DPI crop-detail check; the service recorded target
dimensions of 3508×2481 pixels. A mock-data preview returned a 26,925-byte PDF.
An anonymous direct HTTP `PATCH` to the published definition returned 403; a
follow-up authenticated read confirmed that the award remained published at the
same version with its name unchanged.

## Code and CI evidence

- [PR #143](https://github.com/myota-platform/myota-next/pull/143) merged to
  `testing` as `5519181f`; its Node CI and image-build/scan checks passed.
- [PR #144](https://github.com/myota-platform/myota-next/pull/144) pins the
  award test images by digest and merged to `testing` as `5ed913ce`.
- [PR #160](https://github.com/myota-platform/myota-next/pull/160) adds a
  published-edit scope-denial service test and Admin browser coverage for
  validation and stale-version conflict feedback. It merged as `1d54a05` after
  Node CI and all image-build/scan checks passed.
- The deployed synthetic qualification confirms that a mismatched background
  can be resized for A4 landscape 300 DPI, published as print-ready, and used by
  the preview renderer. An anonymous request cannot change the published record.
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

- The Admin browser editor test uses API fixtures; it does not exercise the
  deployed browser against the current `myota-test` API.
- The synthetic publication verifies the API path, but not a saved published
  revision in the Admin browser with a prior issuance retained. Keep issue #105
  open until that version-history workflow is demonstrated end to end.

Keep the four child issues and issue #104 open until their remaining acceptance
criteria are verified and linked from the issues and Phase 6 qualification
issue #91.
