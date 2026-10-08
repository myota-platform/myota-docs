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
1. Add Grafana Loki in single-binary mode for the current scale.
2. Use pinned image versions consistent with the repository's existing version
   management style.
3. Add persistent Loki storage in Compose and Helm.
4. Initial retention target: 14 days; make storage/retention configurable where
   appropriate.
5. Connect the existing OTel logs pipeline to Loki using OTLP/OTLP HTTP. Keep the
   collector as the single ingestion/control point.
6. Do not add Promtail/Filebeat/Fluent Bit in this phase.
7. Keep the debug exporter only where useful for explicit development/debug use;
   production logging must be durably exported to Loki.
8. Provision a Grafana Loki datasource in both Compose and Helm environments.
9. Keep Loki internal; do not publish an unnecessary public host/Ingress endpoint.
10. Keep Compose and Helm functionally aligned.
11. Add health checks and startup dependencies without introducing dependency
    cycles that prevent services from starting if observability is disabled.
12. Add/update tests or chart-render checks, and verify the observability profile
    locally if the execution environment allows it.
13. Update deployment/operations documentation with storage and retention details.

Do not implement application logging semantics beyond what is strictly required
to prove OTLP log ingestion. Do not implement dashboards or log alerts yet.

Return exact test/render evidence, changed files, operational commands, and any
risks that Phase 5/6 must address.
