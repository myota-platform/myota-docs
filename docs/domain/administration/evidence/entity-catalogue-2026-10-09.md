# Entity catalogue delivery evidence — 9 October 2026

## Implementation and retained evidence

- [Focused entity workspace `5ac4534`](https://github.com/myota-platform/myota-admin-web/commit/5ac4534).
- [Browser regression/CI implementation `2d497e2`](https://github.com/myota-platform/myota-admin-web/commit/2d497e2).
- [Read-only live tool `780dad8`](https://github.com/myota-platform/myota-admin-web/commit/780dad8).
- [Collapsed-map reveal and manual candidate regression `d36a702`](https://github.com/myota-platform/myota-admin-web/commit/d36a702).
- [HTTPS-origin confinement in the live tool `2a63bf0`](https://github.com/myota-platform/myota-admin-web/commit/2a63bf0).
- [Organization roadmap `f8b790c`](https://github.com/myota-platform/.github/commit/f8b790c).
- [Guide, navigation diagram and timeline `c417d3d`](https://github.com/myota-platform/myota-docs/commit/c417d3d).

[CI run 37854330126](https://github.com/myota-platform/myota-admin-web/actions/runs/37854330126)
passed unit tests, TypeScript, build and all five Chromium flows, then published
the admin image. Its `catalogue-browser-evidence` artifact retains desktop,
geometry and mobile screenshots. The fixtures are isolated HTTP responses;
Vue, Leaflet and Geoman are real, and public tile requests are blocked.

The final [origin-confinement build 37869393183](https://github.com/myota-platform/myota-admin-web/actions/runs/37869393183)
also passed the checks and published the latest image.

Local verification passed: 24 unit tests; five browser flows; TypeScript;
production build; `git diff --check`. Dependency audit reported no vulnerabilities
after updating the vulnerable transitive `source-map-js` package to 1.2.2.
The flow suite covers retained focus/batch selection, unsaved navigation/close
guards, partial saves, real vertex controls/replacement drafts, retired geometry,
stacked single/bulk confirmations, bulk completion, inline review decisions,
mobile width and manual candidate drawing/submission.

[Helm rendering run 37853865837](https://github.com/myota-platform/myota-deploy/actions/runs/37853865837)
passed in GitHub Actions; no chart rendering was performed locally. The chart
structure, service ports and API contracts did not need to change. Fleet image
digest records, not a new UI service, perform the rolling update.

## Final Helm deployment

Fleet/Helm deployed chart `myota-0.2.11`, release revision **103**. Fleet is
**1/1 Ready** at [deployment digest commit `1909bde`](https://github.com/myota-platform/myota-deploy/commit/1909bde).
All **21 deployments** have their desired Ready, updated and available replicas;
admin-web is 1/1. Its pod-template image digest matches the recorded digest:

`sha256:39e1a8cde7f7b55d35c897b7b2556bcbaddb325095436cfcf830053482efebcb`.

The [final digest-sync workflow](https://github.com/myota-platform/myota-deploy/actions/runs/37882429186)
succeeded. No Compose/Helm structural change or service migration was required:
the redesigned editor uses the existing API and the same frontend port/image.

## Read-only live checks and limits

The K3s host verified the actual admin ingress over HTTPS:

| Check | Observed result |
| --- | --- |
| `/entity-management` | HTTP 200, `Cache-Control: no-store`, existing referrer policy |
| Identity current-account resource | HTTP 200 |
| Paged entity catalogue | HTTP 200, ten returned / ten total entities |
| Shared categories and location-option tree | HTTP 200 |
| Selected entity and audit resources | HTTP 200 |
| Served JS asset | `/assets/index-Dx2ZmmCV.js`; contains the new dialog, navigation and geometry controls |

Authentication used the authorized remote test-account file only within the
server process. No password or token is retained in this record. These checks
did not save, propose, approve, delete or otherwise mutate live entities.

A desktop live browser run could not be completed: local DNS filtering returned
a redirect to an Infoblox block page rather than the admin application. The
smoke tool now refuses cross-origin navigation and installs its ephemeral token
only on the configured admin origin. This is a network limitation, not a passed
visual or production mutation test. Desktop/mobile screenshots and destructive
flow assertions come from the isolated browser suite; actual live deletion
and editing were deliberately not exercised against the ten real entities.
