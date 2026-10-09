# Programme and award designer delivery — 9 October 2026

## Evidence collected

- TypeScript validation, production build and 26 frontend unit tests passed
  locally. Eleven isolated Chromium flows include programme/award/UTC
  regressions and five existing catalogue/geometry/deletion flows. Publication
  input tests run in a Europe/Madrid browser, not an accidentally UTC browser.
- Activity HTTP and domain tests run inside Python 3.12/Pillow 11.3 on Colima.
  Tests include PNG/JPEG binary content, legacy draft round trips, bounded
  preview rendering and physical A4/Letter portrait/landscape dimensions.
- Browser screenshot inspection confirms populated, aligned programme fields
  and restored editable certificate placement controls.
- OpenAPI mirrors, operation/client coverage and runtime routes are reconciled;
  added routes are documented in the [designer guide](../programme-and-award-design.md)
  and [REST consolidation plan](../api-rest-consolidation-plan.md).

The Colima activity suite passed **26 tests**, including the isolated JetStream
overlap/restart integration test (no skips). Existing deployment runtime tests
passed **32 tests**. Preview PDFs were rendered with Poppler and inspected:
mock callsign/name/date/manager text fits, and A4 landscape has a MediaBox of
841.92×595.68 points. Tests independently verify all four paper/orientation
combinations. Generated QA files are temporary, not credentials or issued PDFs.

Platform integration tests passed 34 checks with one opt-in JetStream-retention
test skipped because that suite was not configured with its isolated broker.
The activity-owned JetStream regression did run and pass in Colima and CI.
After adding UTC normalization, the activity HTTP suite also passed under
`TZ=Pacific/Honolulu` (its opt-in broker test was skipped in that repeat). Seven
programme tests verify offset/legacy normalization and UTC publication/event
payloads under the same non-UTC runtime timezone.
Ruff formatting/lint, pre-commit/pre-push hooks and whitespace checks passed.
Contract reconciliation reports 170 operations and 271 registry routes with
zero missing contract/registration entries or duplicate operation IDs; both
client facades cover all 25 preferred methods checked by CI.

## Implementation commits and CI

| Repository | Commits |
| --- | --- |
| Admin web | [Programme fields `80caaf8`](https://github.com/myota-platform/myota-admin-web/commit/80caaf8), [designer `7d6d5c8`](https://github.com/myota-platform/myota-admin-web/commit/7d6d5c8), [effective-date round trips `f752028`](https://github.com/myota-platform/myota-admin-web/commit/f752028) |
| Activity | [Content, rendering and HTTP tests `f02148f`](https://github.com/myota-platform/myota-activity-service/commit/f02148f) |
| Contracts | [Resources, schemas and clients `0a2d892`](https://github.com/myota-platform/myota-contracts/commit/0a2d892) |
| Platform | [Owner runtime reconciliation `f2ab859`](https://github.com/myota-platform/myota-platform/commit/f2ab859) |
| Deployment | [Runtime synchronization `7d767fa`](https://github.com/myota-platform/myota-deploy/commit/7d767fa) |
| Documentation | [Guide, REST plan and timeline `2e9c447`](https://github.com/myota-platform/myota-docs/commit/2e9c447) |
| Organization profile | [Roadmap `fd594fc`](https://github.com/myota-platform/.github/commit/fd594fc) |

Passed workflows: [final admin build/browser checks](https://github.com/myota-platform/myota-admin-web/actions/runs/37897357925),
[activity tests/image](https://github.com/myota-platform/myota-activity-service/actions/runs/37896758762),
[platform tests](https://github.com/myota-platform/myota-platform/actions/runs/37896760038),
[deployment images](https://github.com/myota-platform/myota-deploy/actions/runs/37896896387),
[contract freeze](https://github.com/myota-platform/myota-contracts/actions/runs/37896901118),
and [GitHub Helm lint/render](https://github.com/myota-platform/myota-deploy/actions/runs/37896915432).
The admin workflow retains programme/designer screenshots in its existing
`catalogue-browser-evidence` artifact. Helm was not rendered locally.

## Live API checks and verification limits

The K3s host authenticated with the authorized remote test-account file; no
credential/token appears in repository artifacts. Both existing programme
details still return their identifier/name through the API. The live deployment
currently has zero award definitions and zero registered artwork, so legacy
award load/save verification uses fixtures rather than claiming an existing
live award was tested.

All four transient preview calls returned HTTP 200 and mock `EA7TEST` data:

| Paper/orientation | PDF MediaBox (points) |
| --- | --- |
| A4 portrait | 595.68 × 841.92 |
| A4 landscape | 841.92 × 595.68 |
| Letter portrait | 612 × 792 |
| Letter landscape | 792 × 612 |

The award-definition count remained zero after previews. Admin `/programmes`
and `/awards` returned HTTP 200. Final image/release verification is recorded
below after Fleet reconciliation.

Local browser regression responses are fixtures; they do not prove live saves.
No existing production programme, award, asset or entity is modified for testing.
Live uploads and draft mutations were not exercised to avoid leaving synthetic
award assets or changing programme policy. Browser popup behavior is verified
in the isolated suite; live PDFs were checked through the actual ingress API.
