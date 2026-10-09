# Programme and award designer delivery — 9 October 2026

## Evidence collected

- TypeScript validation, production build and 24 existing frontend unit tests
  passed locally. Nine isolated Chromium flows include four programme/award
  regressions and five existing catalogue/geometry/deletion flows.
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

CI and live rollout results will be recorded after delivery.
Local browser regression responses are fixtures; they do not prove live saves.
No existing production programme, award, asset or entity is modified for testing.
