# UTC time policy

MyOTA uses **UTC throughout**, following amateur-radio logging practice. A
programme cannot change the operational timezone. Browser locale affects
language/number formatting, not the instant or timezone of displayed times.

## API, storage and execution

- Lifecycle/audit/event timestamps are timezone-aware UTC. API date-times use
  ISO 8601 with `Z` (or an explicit zero offset). Numeric token expiries and
  telemetry timestamps are epoch-based and independent of timezone.
- Activation/QSO parsing normalizes offsets to UTC; ADIF `QSO_DATE`/`TIME_ON`
  are UTC. Award and programme-policy/content effective dates now normalize
  offsets to UTC before persistence/publication and event emission.
- Legacy timezone-less date-times mean UTC, **not** browser/server local time.
  Invalid effective dates are rejected. Old data is not blindly shifted or
  rewritten; offset-qualified values retain the same instant when normalized.
- Date-only achievement/certificate dates derive from the UTC calendar day.
  Retention cutoffs, statistics and worker timestamps already use UTC.
- All three live databases were inspected: core/activity report `UTC`, geo
  reports `Etc/UTC`, and log timezone is likewise UTC. No database migration,
  data deletion or host-wide timezone change is necessary. PostgreSQL
  `timestamptz` stores an instant; keep application/database sessions in UTC.
- Import source provenance remains immutable. An original dataset's declared
  timezone/date text is not rewritten as if it were a platform event timestamp.

## Web clients and observability

The Admin UI labels effective-date inputs **(UTC)** and displays a UTC reminder
in the header. Although the browser control is named `datetime-local` in HTML,
its digits explicitly represent UTC: the client appends `Z` when submitting.
There is no local-to-UTC hour shift. Loading an offset-qualified date converts
its instant to UTC digits; unchanged values retain their seconds/fractions.
Awards, policy and content publication all use the shared UTC client helpers.
NATS and SeaweedFS status/history displays use explicit UTC rather than
`toLocaleString()` with the browser timezone.

The participant client does not currently perform local-time conversion;
timestamps returned by services stay UTC. New participant/mobile screens must
use the same rule, including daylight-saving transitions and midnight rollover.

All five provisioned Grafana dashboards explicitly use `timezone: utc`, retaining
the last 30 minutes and 30-second refresh. Compose and Helm set Grafana's default
timezone to UTC for new/ad-hoc dashboards; administrators should keep copied or
custom dashboards in UTC. Personal Grafana timezone overrides are not removed
or prohibited by this change. Logs and telemetry retain UTC timestamps.
Both Kubernetes retention CronJobs specify `timeZone: Etc/UTC`, so schedules
do not depend on the K3s controller's host timezone.

## Verification and maintenance

- Frontend UTC unit tests cover offset conversion, legacy UTC interpretation,
  untouched seconds, new input serialization and explicit UTC display.
- Chromium tests run with **Europe/Madrid**, checking UTC effective dates in
  award, policy and content screens; results cannot accidentally pass just
  because the browser itself is UTC.
- Programme publication tests run with `TZ=Pacific/Honolulu`, verifying UTC
  persistence and event payloads. Activity HTTP tests verify an offset-bearing
  draft is normalized to the same UTC instant.
- Deployment tests require every provisioned dashboard to match its Helm copy,
  use UTC and retain the requested time/refresh defaults. Helm rendering runs
  in GitHub Actions, not locally.

See [programme/award delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md).
Keep API/model documentation consistent with this policy. Future rule and
statistics-period configuration must use UTC day boundaries, not programme
timezone settings.
