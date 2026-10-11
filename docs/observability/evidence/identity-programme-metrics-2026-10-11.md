# Identity and Programme metrics coverage — 11 October 2026

## Result

Identity and Programme now expose bounded HTTP request counters and duration
histograms at their existing `/metrics` endpoints. Deploy scrapes them as
separate named OpenTelemetry Collector receiver jobs, and the existing API
performance dashboard queries both through the `service_name` resource label.
The three metrics-coverage issues are complete: [Identity #4](https://github.com/myota-platform/myota-identity-service/issues/4),
[Programme #3](https://github.com/myota-platform/myota-programme-service/issues/3),
and [Deploy #7](https://github.com/myota-platform/myota-deploy/issues/7).

## Source changes and checks

| Repository | Main commit | Focused verification |
|---|---|---|
| Identity | [`846da38`](https://github.com/myota-platform/myota-identity-service/commit/846da380ce733863fd7710b1994f008aaaa7fa14) | Metric unit tests; Ruff/Python quality [run 38176482981](https://github.com/myota-platform/myota-identity-service/actions/runs/38176482981); immutable image build [run 38176482105](https://github.com/myota-platform/myota-identity-service/actions/runs/38176482105). |
| Programme | [`88b12dd`](https://github.com/myota-platform/myota-programme-service/commit/88b12ddd6e31282ae2e0d8822c471eeb77adc954) | Metric unit tests; Ruff/Python quality [run 38176487850](https://github.com/myota-platform/myota-programme-service/actions/runs/38176487850); immutable image build [run 38176487064](https://github.com/myota-platform/myota-programme-service/actions/runs/38176487064). |
| Deploy | [`26d13ab`](https://github.com/myota-platform/myota-deploy/commit/26d13abaef55f1c545d0fff3b8003f2572bd5483) | Python quality [run 38176708410](https://github.com/myota-platform/myota-deploy/actions/runs/38176708410); Helm chart validation [run 38176707726](https://github.com/myota-platform/myota-deploy/actions/runs/38176707726). |
| Platform mirror | [`45437c6`](https://github.com/myota-platform/myota-platform/commit/45437c64ee4f52559b434ede65797e08b8395ac9) | Mirror CI [run 38176714874](https://github.com/myota-platform/myota-platform/actions/runs/38176714874). |

Metric labels contain only HTTP method, registered route template, and bounded
response status. They exclude account IDs, callsigns, programme identifiers,
raw URLs, exception text, and other unbounded or sensitive values. Service
aggregate gauges remain sourced from each owning service's database.

## Deployed verification

At 2026-10-11 21:36 UTC, Helm release `myota` in namespace `myota` was at
revision 261, status `deployed`. The deployed Identity image was
`sha256:9bb32a202453319863de357d9dc643bdf6a000e1ae3aa615ffa510e3ea06fa2a`;
Programme was `sha256:8a89e03ba1cf8a8044e2f99658e6dd7a0d5a099097e7ef7818cdaa9a4d1938c2`.
Both Deployments were Ready. The full namespace had 23/23 Deployments and 27/27
replicas Ready; the MyOTA Fleet BundleDeployment reported Ready with 67/67
resources.

Read-only GET checks returned HTTP 200 JSON from Identity `/healthz` and
Programme `/healthz`, both with `durable:true`. Both `/metrics` endpoints
returned HTTP 200 and Prometheus text exposition. The live scrape contained
`myota_http_server_requests_total` and
`myota_http_server_duration_milliseconds_count` for each service. Prometheus
instant queries for
`myota_http_server_requests_total{service_name="myota-identity"}` returned
three route/status series; the equivalent Programme selector returned two.
The API performance dashboard's request-rate and p95-latency queries returned
series for both services. The selected requests were health and metrics GETs;
no domain write or test data was created.

## Completion boundary

This closes service metrics instrumentation, scrape configuration, dashboard
query coverage, and the live non-error endpoint/series checks in the three
linked issues. It does not close the separate structured logging acceptance
gates for trace navigation, producer-specific worker correlation, Loki privacy,
alerting/retention/recovery, or full Compose runtime execution.
