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

The subsequent UTC policy supersedes the interim local-display behavior in
`f752028`. Final changes: [Admin `8695c80`](https://github.com/myota-platform/myota-admin-web/commit/8695c80),
[Activity `740e69f`](https://github.com/myota-platform/myota-activity-service/commit/740e69f),
[Programme `c911ccd`](https://github.com/myota-platform/myota-programme-service/commit/c911ccd)
with [corrected policy regression fixtures `59ca65c`](https://github.com/myota-platform/myota-programme-service/commit/59ca65c),
[Platform `2640891`](https://github.com/myota-platform/myota-platform/commit/2640891),
[Contracts `5a2eb03`](https://github.com/myota-platform/myota-contracts/commit/5a2eb03),
[deployment UTC settings `be48315`](https://github.com/myota-platform/myota-deploy/commit/be48315)
and [Helm 0.2.12/render assertions `8413339`](https://github.com/myota-platform/myota-deploy/commit/8413339).
The initial policy-test fixture errors were corrected before the final programme
image could publish; its seven real-handler tests now pass.

Passed UTC workflows: [Admin](https://github.com/myota-platform/myota-admin-web/actions/runs/37898827644),
[Activity](https://github.com/myota-platform/myota-activity-service/actions/runs/37898834055),
[Programme](https://github.com/myota-platform/myota-programme-service/actions/runs/37898892773),
[Platform](https://github.com/myota-platform/myota-platform/actions/runs/37898841452),
[Contracts](https://github.com/myota-platform/myota-contracts/actions/runs/37899098092),
[deployment images](https://github.com/myota-platform/myota-deploy/actions/runs/37899091890)
and [GitHub Helm render with UTC assertions](https://github.com/myota-platform/myota-deploy/actions/runs/37899092076).

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
and `/awards` returned HTTP 200. The final served bundle is
`/assets/index-BP4LcrU0.js`: it includes the preview API, UTC effective-date labels
and the UTC shell reminder. The final live landscape preview returned a valid
PDF and the UTC achievement date; the award-definition count remained zero.

## Final Helm/Fleet delivery

Chart **myota-0.2.12**, Helm release revision **111**, is **deployed**. Fleet
is **1/1 Ready** at [image-digest commit `4a5b650`](https://github.com/myota-platform/myota-deploy/commit/4a5b650).
All **21 deployments** have their desired Ready, updated and available replicas;
all three database StatefulSets are 1/1 and all nine PVCs are Bound. The migration
gate succeeded. Fleet briefly reconciled successive image/chart updates before
reaching its final Ready state; no manual database or entity mutation was used.
The [final digest-sync workflow](https://github.com/myota-platform/myota-deploy/actions/runs/37899257265)
succeeded.

The checked deployment digest annotations match the recorded Helm values:

| Service | Digest |
| --- | --- |
| Admin web | `sha256:e1b12ab3295b5d9d872a9f11514537a327fe8e92eb8f6dea472e5f571ded7c4d` |
| Activity | `sha256:d7967a46485cc232ffd9084e94725442388b1caa3142fb1f3a29d1e0c08eda37` |
| Programmes | `sha256:891c02d3a94ce751a52fdeda90e356d2319639e419642d0ce44686489bd7a765` |
| Gateway | `sha256:f0b7c42bb6a5955f6bb2a68a715fedaddd15610996ea3d79d1607ad0baad2bb4` |

Authenticated Grafana API checks report default timezone **UTC** and explicit
`utc` for all five provisioned dashboards. Every dashboard retains
`now-30m` → `now` and `30s` refresh. Both retention CronJobs report `Etc/UTC`.
The live core/activity database and log timezones are `UTC`; geo uses `Etc/UTC`.

Local browser regression responses are fixtures; they do not prove live saves.
No existing production programme, award, asset or entity is modified for testing.
Live uploads and draft mutations were not exercised to avoid leaving synthetic
award assets or changing programme policy. Browser popup behavior is verified
in the isolated suite; live PDFs were checked through the actual ingress API.
