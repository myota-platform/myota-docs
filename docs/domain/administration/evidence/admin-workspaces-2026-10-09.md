# Administration workspace delivery — 9 October 2026

## Reviewed scope

Reviewed the shell, routes, session initialization, existing workspace views,
shared map/dialog controls and owning service permission checks. The
[review/decisions table](../navigation-reorganization.md#review-and-decisions)
records the changes. No backend privilege, lifecycle or tile-policy expansion.

## Validation and limits

- Type check, production build and **30 frontend unit tests** passed locally.
- All **23 browser tests passed** locally. Existing **11 browser regressions** cover catalogue dialogs, geometry,
  candidates, protected deletion, programme saves, award assets/design/previews
  and UTC publication dates. Twelve additional browser tests cover navigation,
  permissions/deep links, partial failure/retry, user round trips/scoped grants,
  password reset fields, immutable roles, policy/content read-only state,
  programme context, category saves, mobile keyboard navigation, award readers,
  operations refresh and session preservation during API outages.
- Tests use real Vue/Leaflet/Geoman with isolated APIs, block public tiles and
  do not modify live data. Save checks prove UI behaviour and payloads, not
  live database writes. Fixtures and credentials are synthetic.
- Desktop overview/user editor and mobile screenshots are retained in the
  admin workflow's `catalogue-browser-evidence` artifact. Visual review checks
  the existing style, aligned fields, readable navigation and narrow-screen fit.

## Commits and continuous integration

| Repository | Delivery |
| --- | --- |
| Admin web | [Navigation `7f67ef3`](https://github.com/myota-platform/myota-admin-web/commit/7f67ef3), [editors `437ca34`](https://github.com/myota-platform/myota-admin-web/commit/437ca34), [tests and guide `80c4cfb`](https://github.com/myota-platform/myota-admin-web/commit/80c4cfb) |
| Documentation | [Guide, diagram, indexes and changes.md `3b69e60`](https://github.com/myota-platform/myota-docs/commit/3b69e60) |
| Organization profile | [Roadmap `e1eb403`](https://github.com/myota-platform/.github/commit/e1eb403) |
| Deployment | [Published image digest `33ba50d`](https://github.com/myota-platform/myota-deploy/commit/33ba50d) |

Passed: [admin build, unit/types and 23 browser checks](https://github.com/myota-platform/myota-admin-web/actions/runs/37903882227),
[image digest synchronization](https://github.com/myota-platform/myota-deploy/actions/runs/37904140355),
and [GitHub Helm lint/render of the exact deployment commit](https://github.com/myota-platform/myota-deploy/actions/runs/37904376634).
The admin workflow retains screenshots in `catalogue-browser-evidence`.
Helm was not rendered locally.

## Live read-only checks

Chromium connected to the real K3s ingress through an SSH tunnel, keeping the
intended admin origin/SNI to avoid the local DNS filter. The authorized remote
test-account file was used for login; no token/password was saved in evidence
or printed. API writes and public tiles were blocked after authentication.

All fourteen pages loaded the new headings and refresh controls: Overview,
Programmes, Entity categories, Users & access, Rules & policies, Content &
translations, Awards & certificates, Activations & QSOs, Geodata imports,
Entity map, NATS / JetStream, SeaweedFS storage, Entity catalogue and Geodata
review. Existing programme identifier/name and category code loaded correctly.
The catalogue opened existing name/geometry data and a Leaflet map with
attribution; review displayed source comparison.

Result: **0 browser errors, 0 failed API responses, 0 attempted API writes**.
Existing live account edits, category/programme saves, artwork uploads and
entity deletion were deliberately not exercised. Those mutations use isolated
browser round-trip fixtures and existing regression coverage, not a claim of
live persistent writes. Restricted-role live credentials were not created;
those access cases were verified with isolated canonical-account fixtures.

The deployed admin pod is Ready at digest
`sha256:0f680e613bbce2c19bcab73496cec6c11cec9c07b3af03f2cd1a85f4c4a2bb9e`.
All 21 deployments, three database StatefulSets and nine Bound PVCs were
verified after rollout; the final Fleet/Helm release state is recorded below.

Final state: **Helm 0.2.12 revision 114 deployed**, Fleet **1/1 Ready** at
`33ba50d4ee17e17081101731930532e5e86af4bb`. The admin Deployment is **1/1 Ready**
at the published digest above. The existing migration hook completed; no new
schema migration was introduced by this UI delivery. No persistent data or
unrelated cluster workload was changed by the verification.
