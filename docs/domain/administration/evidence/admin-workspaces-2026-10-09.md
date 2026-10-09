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

CI, commit and live Fleet/Helm results will be appended after publishing. Live
checks are read-only; real account edits, uploads and deletions are excluded.
