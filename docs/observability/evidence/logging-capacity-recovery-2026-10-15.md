# Logging capacity and recovery evidence — 15 October 2026

## Scope

This Helm-only qualification advances [logging acceptance #48](https://github.com/myota-platform/myota/issues/48)
under [workstream #57](https://github.com/myota-platform/myota/issues/57).
It ran against the isolated `myota-test` release. No Compose or Podman
deployment was used, and production was not changed.

## Bounded ingestion run

The OpenTelemetry Collector's OTLP/HTTP receiver accepted synthetic log
records at 200 records per second for 600 seconds. The generator submitted
100 records per request over four fixed `service_name` values:
`myota-identity`, `myota-programmes`, `myota-activity`, and `myota-geodata`.
It completed 1,200 batches and 120,000 records with zero sender errors.

Loki's LogQL count over the test interval returned 120,000 records total and
30,000 per service. The indexed label set remained exactly
`deployment_environment`, `service_name`, and `service_namespace`; test data
did not add high-cardinality labels.

## Recovery exercise

Loki was restarted while ingestion continued. Its Helm-managed Deployment
returned Ready 1/1. During the restart, Collector logs showed bounded exporter
retries after connection-refused responses from Loki. After Loki recovered,
the retries stopped, ingestion reached the full expected record count, and a
review of the Collector's last three minutes found zero exporter failures.
The Collector remained Ready 1/1, and Loki `/ready` returned HTTP 200.

At the end of the run, sampled usage was approximately 8m CPU and 139 MiB
memory for Loki (limits: 750m / 768 MiB), and 22m CPU and 85 MiB for the
Collector (limits: 300m / 512 MiB). These are end-of-run samples, not peak
measurements.

## Limits

This establishes a repeatable Testing workload envelope and recovery from a
single Loki restart while logs are being sent. It does not establish production
capacity or multi-node failure tolerance: `myota-test` is a single-node K3s
environment, CPU and memory samples were not continuous peak measurements, and
the test used synthetic rather than representative application traffic. Do not
extrapolate these rates into a production sizing guarantee.

The prior 12 October evidence is retained as a historical snapshot. Its
deployment-test method is superseded by the current Helm-only instruction; no
Compose or Podman test is required or authorized for this workstream.
