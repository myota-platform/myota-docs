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
  Grafana API alert rules at commit
  [7cd6ac2](https://github.com/myota-platform/myota-deploy/commit/7cd6ac233715ceedae25dd049441ed0ce080bee5).
  Platform's synchronized chart copy is at
  [07eeb86](https://github.com/myota-platform/myota-platform/commit/07eeb863330223f426234b32d3078a044d8f65f3).
- Fleet's MyOTA BundleDeployment is Ready with 71/71 resources for Deploy
  commit 7cd6ac2. All 23 Deployments and 27/27 desired replicas are ready.
  Helm revision 278 is currently `pending-upgrade`; revision 277 was
  `deployed`. Fleet reports the applied bundle Ready, but Helm's status
  discrepancy remains recorded rather than described as a successful Helm
  release.
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
high-cardinality labels or log-derived duplicate alerts were added. Fleet
reports the Deploy/Platform rule assets applied.

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
- Helm revision 278 reports `pending-upgrade` while Fleet's applied bundle is
  Ready. This discrepancy must clear before claiming a fully reconciled Helm
  release.

The issue remains open until the browser and Compose gates have real evidence
and the Helm state is reconciled. No Compose run, authenticated UI test, or
production-capacity qualification is claimed here.
