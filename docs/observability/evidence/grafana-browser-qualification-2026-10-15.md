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
- [ ] Follow trace-to-log navigation in the same authorized browser session.

Reverse navigation remains incorrect: **Logs for this span** generated a query
with `|="<trace ID>"`, which searches the plain event body. MyOTA stores that
identifier as structured metadata. The existing log pane still displayed its
record, so its row count must not be treated as success in the new reverse-query
pane. The next correction must use an explicit `trace_id` metadata predicate
and verify the new pane's result, following
[Tempo custom trace-to-log query guidance](https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/configure-trace-to-logs/).

A full dashboard-set comparison also found `myota-k3s-workload.json` and
`myota-outbox-relay.json` only in Helm. The prior source-parity test checked
Compose's subset and therefore did not prove full set parity. This remains a
#22 implementation gate; Kubernetes-only data should remain unavailable in
Compose rather than fabricate measurements.

These remaining gates prevent issue closure.
The browser also observed Grafana Live WebSocket HTTP 400 responses. The
ordinary dashboard and datasource HTTP requests succeeded; streaming Live is
not used to establish log/trace navigation acceptance. Testing Helm revision 332 is now `deployed`; Fleet reports 1/1 ready with
76/76 resources ready at `d225058`. This precedes PR #84 and does not prove its
rollout. Compose runtime/restart parity and representative capacity/recovery qualification remain separate
unfinished #48 gates.

At the post-merge inspection, Helm revision 333 was `pending-upgrade` and
Fleet displayed `WaitApplied` at `cf9daf2`. The Grafana rollout and mounted
source check passed independently; full release reconciliation is not claimed.
