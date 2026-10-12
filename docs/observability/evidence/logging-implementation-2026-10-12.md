# Structured logging and correlation evidence — 2026-10-12

## Result

Identity, Programme, Activity, and Geodata producer-owned correlation issues
are closed with source, CI, deployed-path evidence, and cleanup records.
Deploy issue #6 remains open for authenticated Grafana navigation, representative
capacity/recovery qualification, and full Compose runtime parity. Controlled
single-node probes do not establish representative production behavior.

## Current host deployment

- K3s context: `default`; namespace: `myota`.
- Deploy main is at
  [b460a69](https://github.com/myota-platform/myota-deploy/commit/b460a6900d48b0fe552ca03de21006d812edd824);
  Platform's synchronized chart fixes are in
  [5555541](https://github.com/myota-platform/myota-platform/commit/5555541c8be02891f91afee9c7e0898e9bff428f).
  Deploy Python quality run 38218406097 passed. Helm chart validation
  38217036883 passed for the corrected chart. A focused PyYAML check passed
  all six Prometheus/Grafana rule files across Deploy and Platform.
- Helm revision 280 is `deployed`. Revision 279 is the completed rollback
  to revision 277 source; revision 278 remains in the recorded upgrade history.
  Prometheus and Grafana are 1/1, and all 23 Deployments / 27 desired replicas
  are ready.
- MyOTA's Fleet GitRepo is at commit b460a69, force-sync generation 94, and
  Ready 1/1. Its BundleDeployment is Ready with sync generation 94/94 and
  matching applied/desired deployment IDs. The wider local Fleet cluster shows
  unrelated non-ready bundles; no whole-cluster Ready claim is made.
- The first runbook-link deployment had malformed annotation formatting:
  Prometheus rejected the rule file and restarted. Deploy commit
  [4910201](https://github.com/myota-platform/myota-deploy/commit/4910201b4462aef07e9b55cc35158341ef599ef1)
  fixed Prometheus YAML; Deploy commit
  [b460a69](https://github.com/myota-platform/myota-deploy/commit/b460a6900d48b0fe552ca03de21006d812edd824)
  fixed the separate Grafana alert YAML. Helm rollback restored Prometheus
  before Fleet applied the corrected revision.

## Live alert and dashboard assets

- `promtool check rules /etc/prometheus/rules.yml` inside the Prometheus pod
  returned `SUCCESS: 33 rules found`. All 33 have a runbook URL.
- The running Grafana pod's mounted `alert-rules.yml` contains two
  `runbook_url` entries.
- The running Grafana pod mounts a Logs / Failures link in
  `myota-api-performance.json`,
  `myota-application-workflows.json`, and
  `myota-geodata-capacity.json`. Grafana logged
  `finished to provision dashboards`.
- Alert thresholds and metric ownership remain unchanged. No log-derived
  duplicate alerts or high-cardinality labels were added.

These checks verify that alert and dashboard assets are loaded and provisioned.
They do not verify authenticated trace-to-log or log-to-trace navigation in a
browser.

## Correlation and worker evidence

The Activity consumer reads message metadata through `Msg.headers`; the
redelivery/idempotency regression passes. Activity image
`sha256:825237102efed92775500b1293e3386b3b1a79730bb2e1d07e24c62e659aaaac`
is deployed. Python quality run 38214601936 and image build run 38214600992
passed.

Deploy commit
[9bb3b16](https://github.com/myota-platform/myota-deploy/commit/9bb3b16ce7fb75b5084be2ec6180e5b5b4a8e671)
and Platform mirror
[d110630](https://github.com/myota-platform/myota-platform/commit/d1106301e6ad887a7a3ca040fd7faba137efb104)
set production resource identity and OTLP export for Activity and Geodata
workers, Activity notifications, and three outbox relays. Sanitized Activity
and Geodata producer probes completed through their workers with matching
correlation in Loki; durable ACK floors advanced and pending/ack-pending
counts cleared. Exact application rows, objects, jobs, and test messages were
removed. Programme's fact reached `MYOTA_EVENTS` with request correlation;
the contract has no Programme consumer group, so a worker ACK does not apply.

Detailed source, CI, image, ACK, and cleanup evidence is in closed
[Identity #3](https://github.com/myota-platform/myota-identity-service/issues/3),
[Programme #4](https://github.com/myota-platform/myota-programme-service/issues/4),
[Activity #3](https://github.com/myota-platform/myota-activity-service/issues/3),
and [Geodata #6](https://github.com/myota-platform/myota-geodata-service/issues/6).

## Loki privacy, labels, and retention

- Loki's live `/loki/api/v1/labels` endpoint reports exactly three indexed
  labels: `deployment_environment`, `service_name`, and
  `service_namespace`. Correlation IDs, trace IDs, request IDs, normalized
  routes, and other event attributes remain structured metadata, not index
  labels.
- Sampled live Activity, Geodata, and relay records contain bounded operational
  metadata but no request body or domain payload. The event name is the log
  body.
- A sanitized Geodata 403 probe included a unique marker in a path segment.
  Loki showed the normalized route and status metadata; the marker was absent.
  No application row, object, job, or message was created. The probe log itself
  remains diagnostic telemetry subject to Loki retention.
- The live Loki `/config` reports `retention_enabled: true` and
  `retention_period: 2w`. Compactor metrics showed 119 successful
  retention-apply operations. This verifies configuration and compactor
  activity; no actual 14-day expiry claim is made.
- Observed records are a small single-node sample, not representative traffic.
  No capacity conclusion is inferred.

## Compose runtime qualification attempt

The authoritative Deploy `compose.images.yaml` at
[85978c2](https://github.com/myota-platform/myota-deploy/blob/main/compose.images.yaml)
rendered with Podman Compose 1.0.6. It contains the service and messaging image
set but omits Loki, Tempo, Prometheus, Grafana, and the Collector, so that file
alone cannot establish logging-stack Compose parity.

A disposable isolated project was created with loopback-only published ports
and separate named volumes. Its first startup failed because the chosen host
port `127.0.0.1:15432` was already occupied. Podman Compose then continued
creating dependent containers despite the database startup failure, so this
was not a valid runtime or restart check. The project was stopped and cleaned:
test containers, volumes, network, cached images, temporary files, and the
installed Podman packages were removed. No migration job completed and no
application test data was created.

The authoritative full source Compose file is
[`compose.yaml`](https://github.com/myota-platform/myota-deploy/blob/main/compose.yaml)
at [422df5f](https://github.com/myota-platform/myota-deploy/blob/main/compose.yaml).
The local workspace copy still contains the retired Operations workload, so it
was not used for a full run. Compose runtime parity remains unverified; this
attempt does not satisfy the acceptance gate.

## Remaining Deploy issue #6 gates

- Exercise authenticated Grafana trace-to-log and log-to-trace navigation in
  a real authorized browser session.
- Qualify representative capacity assumptions and operational recovery.
- Run full Compose runtime parity and restart checks. Docker and Podman are
  absent on this host; no YAML render is described as runtime parity.

No authenticated browser session, Compose runtime, or representative load test
is claimed.
