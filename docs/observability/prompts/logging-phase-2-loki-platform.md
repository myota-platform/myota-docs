# Phase 2 prompt - Loki platform integration

Use this prompt in ChatGPT Work or Codex to implement Phase 2 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Implement the MyOTA Loki logging backend while preserving the existing
Prometheus + Tempo + Grafana + OpenTelemetry Collector architecture.

Primary repository:
- myota-platform/myota-deploy

Documentation repository:
- myota-platform/myota-docs

Inspect the current Compose, image-based Compose, Helm chart, OTel Collector
configuration, Grafana provisioning and observability documentation before
making changes.

Requirements:
1. Add Grafana Loki in single-binary mode for the current scale, using TSDB
   with the dedicated SeaweedFS S3 bucket `myota-loki` as the durable store for
   chunks and index.
2. Use pinned image versions consistent with the repository's existing version
   management style.
3. Inspect `myota-deploy` and its Helm chart for the existing SeaweedFS S3
   endpoint, bucket provisioning, and credentials/Secret injection patterns.
   Reuse those patterns for Loki; do not hard-code credentials or create a
   parallel credential mechanism.
4. Keep only a small local writable working volume for Loki's active TSDB/WAL,
   index, cache, and compactor working data. Do not use a 10-20 GiB Loki data
   PV; SeaweedFS is the durable log store.
5. Set the initial retention target to 14 days and configure Loki's compactor to
   own retention and deletion of expired log objects.
6. Do not add a separate MyOTA or SeaweedFS cleanup worker, bucket lifecycle
   deletion policy, or other external retention mechanism for Loki data.
7. Connect the existing OTel logs pipeline to Loki using OTLP/OTLP HTTP. Keep the
   collector as the single ingestion/control point.
8. Do not add Promtail/Filebeat/Fluent Bit in this phase.
9. Keep the debug exporter only where useful for explicit development/debug use;
   production logging must be durably exported to Loki.
10. Provision a Grafana Loki datasource in both Compose and Helm environments.
11. Keep Loki internal; do not publish an unnecessary public host/Ingress endpoint.
12. Keep Compose and Helm functionally aligned, including object-store settings,
    secret references, compactor retention, and local working storage.
13. Add health checks and startup dependencies without introducing dependency
    cycles that prevent services from starting if observability is disabled.
14. Add/update tests or chart-render checks, and verify the observability profile
    locally if the execution environment allows it. Check Loki configuration,
    bucket/credential references without exposing secret values, and Compose/
    Helm parity.
15. Update deployment/operations documentation with the `myota-loki` durable
    storage design, small local working-volume purpose, and compactor-owned
    retention.

Do not implement application logging semantics beyond what is strictly required
to prove OTLP log ingestion. Do not implement dashboards or log alerts yet.

Return exact test/render evidence, changed files, operational commands, and any
risks that Phase 5/6 must address.
