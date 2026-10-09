# Phase 1 prompt - Shared structured logging foundation

Use this prompt in ChatGPT Work or Codex to implement Phase 1 of the
[structured logging plan](../logging-implementation.md).

## Prompt

You are implementing Phase 1 of MyOTA structured logging.

Repositories to inspect before changing code:
- myota-platform/myota-identity-service
- myota-platform/myota-programme-service
- myota-platform/myota-geodata-service
- myota-platform/myota-activity-service
- myota-platform/myota-deploy
- myota-platform/myota-docs

Read the current observability implementation and preserve existing metrics and
tracing behavior.

Goal: create one consistent structured logging foundation for MyOTA services.

Requirements:
1. Use Python standard logging plus OpenTelemetry logging/OTLP integration; do not
   introduce Promtail, Filebeat, Fluent Bit, Elasticsearch or OpenSearch.
2. Reuse the same OTel Resource semantics as traces/metrics:
   service.name, service.namespace=myota, deployment.environment,
   service.instance.id.
3. Every structured record must support timestamp, severity, event, component,
   trace_id, span_id, request_id and correlation_id when context exists.
4. Define helpers for bounded structured context rather than string-concatenated
   messages.
5. Provide centralized redaction/safe-field behavior. Never log Authorization,
   JWTs, passwords, signing keys, API keys, S3 credentials, DSNs, complete ADIF
   contents, complete geodata payloads or unrestricted request bodies.
6. Avoid high-cardinality data as logger/resource attributes that will later
   become Loki labels.
7. Preserve graceful operation when the OTel collector is unavailable.
8. Reduce duplicated observability setup across repositories where practical,
   but do not create a packaging/deployment dependency that makes services unable
   to build independently.
9. Add automated tests covering structured output, trace/span enrichment,
   correlation fields and secret redaction.
10. Update relevant README/documentation files.

Before coding, inspect the current copies of common.py and otel.py and decide the
smallest maintainable reuse strategy. Implement it, run the available tests and
quality checks, and report changed repositories/files, test evidence, remaining
risks and any follow-up needed by later phases.

Do not implement Loki infrastructure, Grafana dashboards, or NATS propagation in
this phase.
