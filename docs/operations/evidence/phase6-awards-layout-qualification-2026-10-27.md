# Phase 6 award layout and print qualification

## Scope

This record tracks the published award editor, Award Text, print dimensions,
and background resizing delivered for
[issue #104](https://github.com/myota-platform/myota-next/issues/104)
and child issues [#105](https://github.com/myota-platform/myota-next/issues/105),
[#106](https://github.com/myota-platform/myota-next/issues/106),
[#107](https://github.com/myota-platform/myota-next/issues/107), and
[#108](https://github.com/myota-platform/myota-next/issues/108).

## Deployed test release

At qualification time, the Fleet-managed `myota-test` Helm release was healthy:

- Chart: `myota-next-1.1.6-testing.29a54b7f45ea.1`
- App version: `testing-39cb632`, Helm revision 44
- All chart-managed Deployments were ready
- `/healthz` and `/readyz`: HTTP 200
- Admin and Participant web roots: HTTP 200
- Activity image: `sha256:0d05c9b98ab20fe357d81090db30a5a0d88db1e9a9cc54e9394db4438e3c39da`

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

## Published revision and prior issuance

After PR #161 fixed revision identity allocation, a synthetic test award was
published as version 1 for `part_jud_es`. The existing synthetic Participant had
four qualifying QSOs; the test award's first level evaluated as eligible. A
synthetic signature asset was registered, the award was requested and issued,
and the Activity worker rendered a PDF. The request was seeded through the
Admin-scoped Activity API for this qualification; this step does not verify a
Participant-facing award request screen.

Using the deployed Admin browser, I opened the published version, selected
“Create editable version,” changed Award Text, and saved. The browser submitted
the source award ID and expected version; Activity returned a distinct version
2 draft linked to the source. After a browser reload, the revised Award Text was
still present. The Admin UI then submitted, approved, and published version 2.

The earlier issuance remains linked to version 1, retaining its original Award
Text snapshot. Its PDF remains downloadable, begins with the PDF signature, and
has the same SHA-256 after version 2 was published:
`550c656e302efe28fff7710282a8f7b063ce809351d052a7a313d344fd8cedac`.
An issuance from version 2 retained the revised Award Text and produced a
different downloadable PDF with SHA-256
`13ebcc5ab67073d8d29ab600f6be028303f480349f0e594d84d16cc88294281a`. A preview
request also returned a valid PDF whose mock data contained the revised text.

## Code and CI evidence

- [PR #143](https://github.com/myota-platform/myota-next/pull/143) merged to
  `testing` as `5519181f`; its Node CI and image-build/scan checks passed.
- [PR #144](https://github.com/myota-platform/myota-next/pull/144) pins the
  award test images by digest and merged to `testing` as `5ed913ce`.
- [PR #160](https://github.com/myota-platform/myota-next/pull/160) adds a
  published-edit scope-denial service test and Admin browser coverage for
  validation and stale-version conflict feedback. It merged as `1d54a05` after
  Node CI and all image-build/scan checks passed.
- [PR #161](https://github.com/myota-platform/myota-next/pull/161) fixes
  published revision ID allocation when Admin includes the source `awardId`.
  Node CI and all image-build/scan checks passed; it merged to `testing` as
  `39cb632`.
- [PR #162](https://github.com/myota-platform/myota-next/pull/162) pins the
  deployed Activity image to its immutable digest. Node CI and all image
  build/scan checks passed; it merged to `testing` as `29a54b7`.
- The deployed synthetic qualification confirms that a mismatched background
  can be resized for A4 landscape 300 DPI, published as print-ready, and used by
  the preview renderer. An anonymous request cannot change the published record.
- The protected Rancher upgrade workflow deployed chart
  `myota-next-1.1.6-testing.29a54b7f45ea.1`; the Activity image is pinned by
  digest to the image built from `testing-39cb632`.
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

## Remaining scope limitation

- The saved revision, review, approval, publication, reload, prior issuance,
  and rendered PDF were verified in the deployed test environment. This closes
  the evidence gap for issue #105.
- A Participant-facing awards request screen was not exercised by this
  qualification; the synthetic requests were created with an Admin-scoped
  Activity API token. This remains a separate Phase 6 workflow qualification.

Keep parent issue #104 and remaining child issues #106–#108 open until their
remaining acceptance criteria are verified and linked from the issues and
Phase 6 qualification issue #91.
