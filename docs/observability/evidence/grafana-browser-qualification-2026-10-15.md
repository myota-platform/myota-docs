# Grafana browser qualification — 15 October 2026

## Scope

This record advances [dashboard #22](https://github.com/myota-platform/myota/issues/22)
and [logging acceptance #48](https://github.com/myota-platform/myota/issues/48)
under [workstream #57](https://github.com/myota-platform/myota/issues/57).
The inspection ran after local midnight on 15 October (UTC remains 14 October).
It used the real isolated `admin.test.myota.top/observability/` Grafana proxy
and a headless Chromium browser. Production was not changed.

## Findings and correction

Authenticated browser requests loaded the Logs / Failures dashboard and its
four configured datasources. Existing application telemetry demonstrated three
faults; monorepo [PR #84](https://github.com/myota-platform/myota/pull/84) fixes
all three in the canonical Compose and Helm provisioning copies:

- Helm substituted the Kubernetes namespace `myota-test` into
  `service_namespace`, although applications emit the fixed telemetry resource
  `service.namespace=myota`. That filter hid existing Test failures. The
  dashboard now uses the application namespace; Kubernetes metric namespace
  substitutions remain separate.
- The runtime environment label is `test`; the bounded selector omitted it.
  HTTP warning records use OTLP severity `WARN`, while worker warnings can use
  `WARNING`. The selectors/queries now include both observed values, and counts
  group by `severity_text` rather than an absent `severity` field.
- Expanding a correlated Identity error log exposed a **View trace** link whose
  Tempo query was empty. Grafana provisioning consumed `${__value.raw}` as an
  environment variable. Both provisioning files now use `$${__value.raw}` so
  browser interpolation receives the trace ID. This follows
  [Grafana's provisioning example](https://grafana.com/docs/grafana/latest/datasources/loki/configure/).

## Source checks

Fourteen focused tests passed with:

```sh
PYTHONPATH=deploy/services /root/myota-platform/.venv-phase5/bin/python   -m unittest discover -s deploy/tests -p test_observability.py -v
/root/myota-platform/.venv-phase5/bin/ruff check   deploy/tests/test_observability.py
git diff --check
```

The tests cover source/Helm parity, bounded variables, fixed application
namespace, severity handling, escaped datasource interpolation and the existing
operational dashboard/provisioning boundaries. The original Trace dashboard
matched its mounted file at SHA-256
`510f978c3a78a571b4082dc34894167bd5da97993ddf78b1425ad9236fb8de8d`.

## Browser evidence and remaining gates

The browser used a short-lived Identity-signed access token for an existing
active operator, with authorization validated by the current Gateway. The
inspection did not create a user, refresh session, account fixture, NATS
command or telemetry fixture. It used existing logs and traces only. Tokens
and raw query responses used temporary local files with restricted permissions;
they were removed after inspection.

- [x] Authenticated Grafana dashboard and datasource access.
- [x] Existing sanitized failure log displayed in Explore with a trace ID;
  the corrected dashboard shows HTTP failure and worker failure rows over
  the last 24 hours.
- [x] Existing Tempo failed-trace search returned service/duration context;
  the browser loads both Failed traces and Slow traces over 1 second panels.
- [x] Identify and implement fixes for incorrect filtering and empty trace link.
- [x] All required checks passed; PR #84 merged into `testing` at
  `cf9daf2406c6ce20b6adf80fd0423ece7c257e76`.
- [x] Corrected Logs dashboard mounted in Testing with SHA-256
  `fe4e61ebf4abdade5e42575df40db7932fc20e7123282a142cd39617b33905d9`;
  Grafana rollout completed after PR #84.
- [x] Clicking **View trace** opens the matching Identity trace; the trace
  detail shows `myota-identity: GET /v1/identity/me` and its duration.
- [x] Follow trace-to-log navigation in the same authorized browser session;
  PR #85 uses a service-scoped metadata predicate and the new pane contains
  the matching retained log.

Before PR #85, reverse navigation was incorrect. **Logs for this span**
generated a query
with `|="<trace ID>"`, which searches the plain event body. MyOTA stores that
identifier as structured metadata. The existing log pane still displayed its
record, so its row count must not be treated as success in the new reverse-query
pane. PR #85 implements an explicit `trace_id` metadata predicate; the new
pane's
result is verified below, following
[Tempo custom trace-to-log query guidance](https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/configure-trace-to-logs/).

A full dashboard-set comparison also found `myota-k3s-workload.json` and
`myota-outbox-relay.json` only in Helm. The prior source-parity test checked
Compose's subset and therefore did not prove full set parity. PR #85 adds both
files and a recursive full-set parity test; actual Compose runtime provisioning
remains a gate. Kubernetes-only data remains unavailable in standalone Compose
rather than fabricate measurements.

Full Compose runtime/restart and capacity/recovery gates prevent final
workstream closure.
The browser also observed Grafana Live WebSocket HTTP 400 responses. The
ordinary dashboard and datasource HTTP requests succeeded; streaming Live is
not used to establish log/trace navigation acceptance. In the pre-PR #84
inspection, Helm revision 332 was `deployed` and Fleet reported 1/1 ready with
76/76 resources ready at `d225058`. This precedes PR #84 and does not prove its
rollout. Compose runtime/restart parity and representative capacity/recovery
qualification remain separate
unfinished #48 gates.

At the post-merge inspection, Helm revision 333 was `pending-upgrade` and
Fleet displayed `WaitApplied` at `cf9daf2`. The Grafana rollout and mounted
source check passed independently; full release reconciliation is not claimed.

## Reverse navigation and full source parity — PR #85

PR [#85](https://github.com/myota-platform/myota/pull/85) merged into `testing`
at `9f216240bf2cb4171cb41d62815df45ac09e9445` after every reported final-head
check passed. Sixteen focused observability tests passed. The inventory now
compares every dashboard UID/definition recursively, including nested NATS/K3s
assets: all 21 definitions match across Compose and Helm. Compose provisions
K3s in the K3s folder; its Kubernetes data is explicitly unavailable when the
standalone runtime has no kube-state-metrics source. This is source parity,
not full Compose runtime qualification.

Tempo uses a custom query with mapped service/environment selectors and
`trace_id` structured-metadata filtering. Dollar macros are escaped during
provisioning. The mounted Helm datasource file matched SHA-256
`56bf39f9a15477a9a92772ac694f7211462a9ced4350e3192e83ca3b539d7fd8`;
Grafana rolled out ready at 1/1.

At **22:18:55 UTC on 14 October** (15 October locally), authenticated Chromium
verified the retained Identity HTTP error used in the previous forward check.
Expanding the log and clicking View trace opened the matching Identity span.
Clicking Logs for this span generated a service-scoped metadata query; the new
pane contained its matching record. The verifier also executed that exact
reverse query with the generated time range through the authenticated Grafana
proxy and confirmed the same trace ID in returned metadata. Both panes had a
rendered HTTP log row, so the old pane alone did not satisfy the assertion.
The deployed Tempo datasource API reported `customQuery=true`.

Two earlier attempts dynamically selected other existing records and timed
out waiting for a trace/span control. The final check used the previously
confirmed retained trace. This proves the configured navigation path for that
record; it does not certify that every stored trace is complete or renderable.
No synthetic telemetry or account was created. Temporary token/script files
were removed after the check.

Full Compose runtime/restart, representative capacity, operational recovery,
and retention-expiry qualification remain separately open. Do not mark these
complete from matching JSON, a mounted configuration, or one browser sample.

## Helm runtime dashboard follow-up — 15 October

PR [#87](https://github.com/myota-platform/myota/pull/87) merged to `testing`
at `a5faedf72728783c87f2265a2706f63b6a156229` after all final-head checks
passed. It adds `myota-outbox-relay.json` to the chart's observability
ConfigMap and Grafana mounts; a regression check now asserts both entries.
The focused observability suite passed 17 tests, Helm lint/render passed, and
CI deployment checks passed.

Fleet fetched the merged Testing commit and reports the `myota-test`
BundleDeployment Ready. Grafana rolled out Ready 1/1, and its startup logs
report dashboard provisioning finished. The chart source, ConfigMap key, and
mounted `/etc/grafana/dashboards/myota-outbox-relay.json` file all have SHA-256
`420ab9e3d77e09a120fa0b4fb704c90c02befc226ea3147b28b6a0d8f0f79c26`. All 24
Deployments and all 3 StatefulSets are ready.

Helm still reports revision 451 as `pending-upgrade` (`Preparing upgrade`),
although Fleet reports the BundleDeployment Ready and the updated dashboard is
running. Full Helm release reconciliation is therefore not claimed; issue #22
remains open until the release reports a completed state. No Compose or Podman
deployment checks were run. The existing source-parity unit checks compare the
Compose and Helm dashboard definitions without starting Compose.
