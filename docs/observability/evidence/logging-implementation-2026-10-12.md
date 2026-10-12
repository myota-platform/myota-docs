# Structured logging and correlation evidence — 2026-10-12

## Result

The Identity, Programme, Activity, and Geodata producer/consumer correlation
issues are closed with source, CI, live deployment, and cleanup evidence.
Deploy issue #6 remains open for authenticated Grafana navigation and a full
Compose runtime check. Single-node observations below do not qualify production
capacity or representative traffic.

## Deployed state

- K3s context: `default`; namespace: `myota`.
- Deploy main contains runbook links for all 33 Prometheus rules and both
  Grafana API alert rules. The source change is in
  [commit 7cd6ac2](https://github.com/myota-platform/myota-deploy/commit/7cd6ac233715ceedae25dd049441ed0ce080bee5),
  with malformed YAML corrected in
  [commit 4910201](https://github.com/myota-platform/myota-deploy/commit/4910201b4462aef07e9b55cc35158341ef599ef1).
  Platform's synchronized chart copy is at
  [commit 5555541](https://github.com/myota-platform/myota-platform/commit/5555541c8be02891f91afee9c7e0898e9bff428f).
  API, workflow, and Geodata capacity dashboards link to Logs / Failures in
  Deploy [commit c1d2d31](https://github.com/myota-platform/myota-deploy/commit/c1d2d313e2c7f137c4c67884fe7d1efe9aa70e80)
  and Platform [commit cc7aa3d](https://github.com/myota-platform/myota-platform/commit/cc7aa3d08b8df4983b00ef3b10ebfad24766bf19).
- Helm revision 279 is `deployed` after rollback to revision 277 source.
  Helm revision 278 remains `pending-upgrade` in history. Prometheus is 1/1;
  all 23 Deployments and 27/27 desired replicas are ready.
- Fleet has Deploy commit 4910201 staged, but the BundleDeployment applied
  deployment ID and sync generation 92 remain behind the desired ID/generation
  93. The current live ConfigMap contains the valid pre-change rules; the new
  alert runbook annotations and dashboard links are not yet live.
- Activity's fixed JetStream consumer reads NATS message headers through
  `Msg.headers`, has a passing regression for redelivery/idempotency, and is
  deployed at image
  `sha256:825237102efed92775500b1293e3386b3b1a79730bb2e1d07e24c62e659aaaac`.
  Python quality run 38214601936 and image build run 38214600992 passed.
- Deploy's worker logging configuration now sets production resource identity
  and OTLP export for Activity and Geodata workers, Activity notifications,
  and the three outbox relays. Live Activity and Geodata work probes each
  produced `job.received`, `job.started`, and `job.completed` records with
  stable correlation in Loki. The owning JetStream ACK floors advanced, with
  no pending or ack-pending probe message after completion.
- Programme facts publish their event with the request correlation ID. The
  registry declares no Programme consumer group, so no worker ACK is expected.

Detailed request, image, workflow, durable, and cleanup values are recorded in
the closed service issues:
[Identity #3](https://github.com/myota-platform/myota-identity-service/issues/3),
[Programme #4](https://github.com/myota-platform/myota-programme-service/issues/4),
[Activity #3](https://github.com/myota-platform/myota-activity-service/issues/3),
and [Geodata #6](https://github.com/myota-platform/myota-geodata-service/issues/6).

## Loki privacy, labels, and retention

- Loki's live `/loki/api/v1/labels` endpoint reports exactly three indexed
  labels: `deployment_environment`, `service_name`, and
  `service_namespace`. Correlation IDs, trace IDs, request IDs, route, and
  other event attributes are queryable structured metadata and are not index
  labels.
- Sampled live Activity, Geodata, and relay records contained operational
  identifiers and bounded event metadata, but no request body or domain payload.
  The log body is the event name. No unique IDs were promoted into indexed
  labels.
- A sanitized Geodata 403 probe carried a unique marker in a path segment.
  The Collector-to-Loki record contained the normalized route template and
  status metadata; the marker itself was absent. No application row, object,
  job, or message was created. Loki's retained log is diagnostic telemetry
  subject to the configured retention policy.
- Loki's live `/config` reports `retention_enabled: true` and
  `retention_period: 2w`. Its metrics showed 119 successful retention-apply
  operations. This verifies configuration and compactor activity; it does not
  claim that a newly written record was observed expiring after 14 days.
- The sample reflects current single-node host traffic and controlled probes,
  not representative production load. No capacity conclusion is inferred from
  the small sample.

## Alert/runbook coverage

All 33 Prometheus rules and both Grafana API alerts link to the shared
[logging runbook](../logging-runbook.md#alert-response). Thresholds remain
bounded and continue to use Prometheus metrics where a metric exists; no
high-cardinality labels or log-derived duplicate alerts were added. The
Deploy/Platform assets are committed and syntax-checked; Fleet has not applied
the new versions yet.

## Remaining Deploy issue #6 gates

- Authenticated Grafana trace-to-log and log-to-trace navigation has not been
  exercised in the browser using a real authorized session. Provisioned
  datasource links are present, but config presence is not UI acceptance.
- Docker and Podman are not installed on this host, so the full Compose
  runtime/restart/retention exercise was not run. YAML rendering is not
  represented as runtime parity.
- The current single-node measurements do not establish representative
  capacity or operational recovery under load. Actual 14-day expiry was not
  waited out; the live policy and successful compactor runs were checked.
- Helm revision 279 is `deployed` after rollback to revision 277 source;
  revision 278 remains `pending-upgrade` in history. Fleet is `WaitApplied`
  at commit b460a69; it has not applied the current chart source.

The issue remains open until the browser and Compose gates have real evidence
and the Helm state is reconciled. No Compose run, authenticated UI test, or
production-capacity qualification is claimed here.

## Deployment correction after the initial 12 October check

The first runbook-annotation source edit merged `runbook_url` and the
following `description` on one YAML line. GitHub Helm rendering accepted the
chart, but live Prometheus rejected `observability/rules.yml` and entered
CrashLoopBackOff. The malformed Prometheus rule file was corrected in Deploy commit
[4910201](https://github.com/myota-platform/myota-deploy/commit/4910201b4462aef07e9b55cc35158341ef599ef1)
and Platform commit
[5555541](https://github.com/myota-platform/myota-platform/commit/5555541c8be02891f91afee9c7e0898e9bff428f).
The Grafana alert file's equivalent formatting correction is Deploy commit
[b460a69](https://github.com/myota-platform/myota-deploy/commit/b460a6900d48b0fe552ca03de21006d812edd824).
Deploy Python quality runs 38217037536 and 38218406097 and Helm chart validation
run 38217036883 passed. A focused PyYAML check passed all six Prometheus/Grafana
rule files across source and chart mirrors (33/33 and 2/2 rules have runbook
links). The three dashboard JSON assets parse and contain the Logs / Failures
link.

Helm rollback to revision 277 source completed as revision 279. Prometheus
returned to 1/1 Running and all 23 Deployments / 27 replicas are ready. The
corrected source commit is staged in Fleet, but the BundleDeployment still
reports the prior applied deployment ID and sync generation 92 versus desired
94. The MyOTA GitRepo is at commit b460a69 and reports `WaitApplied` (0/1). Do not claim the runbook/dashboard assets are live until Fleet finishes
that sync. The authenticated Grafana browser check and full Compose runtime
exercise remain open; Docker and Podman are absent on the host.
