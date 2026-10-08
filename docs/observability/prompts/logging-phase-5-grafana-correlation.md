# Phase 5 prompt - Grafana logs and cross-signal correlation

Use this prompt in ChatGPT Work or Codex to implement Phase 5 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Integrate MyOTA Loki logs into Grafana so operators can navigate between metrics,
traces and logs.

Primary repository:
- myota-platform/myota-deploy

Documentation:
- myota-platform/myota-docs

Prerequisites: Loki is receiving structured logs and services emit trace_id,
service identity, component and correlation_id.

Requirements:
1. Provision the Loki datasource for Compose and Helm using stable datasource UID
   myota-loki.
2. Configure a derived field/link so trace_id in Loki opens the corresponding
   trace in the existing Tempo datasource.
3. Configure Tempo trace-to-logs so an operator can open logs relevant to a trace
   or span, scoped at minimum by service and time range.
4. Keep service/environment/severity/component filters low-cardinality.
5. Add a provisioned MyOTA Logs / Failures dashboard with:
   - log events per minute;
   - ERROR/WARN rate;
   - errors by service/component;
   - recent failures table;
   - worker retries/failures;
   - HTTP 5xx log stream;
   - variables for environment, service, severity and component.
6. Provide an easy LogQL path to filter by correlation_id without making it a
   Loki label.
7. Add links from existing operations/API performance dashboards where they
   materially improve troubleshooting.
8. Do not replace Prometheus panels with log-derived approximations when metrics
   already exist.
9. Verify provisioning in Compose and Helm rendering.
10. Document example workflows:
    metric alert -> trace -> logs;
    error log -> trace;
    correlation_id -> multi-service workflow investigation.

Return screenshots only if practical; always provide query/provisioning evidence
and exact changed files.
