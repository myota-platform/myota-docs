# Phase 3 prompt - HTTP logging and distributed trace propagation

Use this prompt in ChatGPT Work or Codex to implement Phase 3 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Implement structured HTTP request logging and standards-based distributed trace
propagation across MyOTA HTTP services.

Repositories:
- myota-platform/myota-identity-service
- myota-platform/myota-programme-service
- myota-platform/myota-geodata-service
- myota-platform/myota-activity-service
- any gateway repository/source used by myota-deploy if trace forwarding is owned there

Prerequisite: preserve and use the Phase 1 structured logging foundation.

Requirements:
1. Replace the current isolated/manual server-span behavior with correct W3C
   Trace Context extraction/injection using traceparent and tracestate.
2. Preserve X-Request-ID and X-Correlation-ID behavior. Generate them when absent
   and return them in responses.
3. Emit one structured http.request.completed record per handled request.
4. Include normalized http.route, method, response status, duration_ms,
   request_id, correlation_id, trace_id and span_id.
5. Do not use raw URL paths containing identifiers as bounded route dimensions.
6. Define sensible severity: server failures ERROR; expected authorization and
   normal not-found outcomes should not flood WARN/ERROR logs.
7. Ensure 400/403/404/500 paths and unexpected exceptions create useful,
   sanitized records.
8. Never log Authorization headers or request bodies by default.
9. Preserve current Prometheus metrics and Tempo traces.
10. Ensure outgoing service-to-service requests propagate trace context and
    correlation ID where applicable.
11. Add tests for newly generated IDs, preserved incoming IDs, route
    normalization, traceparent continuation, 2xx/4xx/5xx completion logs and
    absence of secrets.
12. Run each affected repository's tests/quality checks and document evidence.

Do not implement NATS/JetStream context propagation in this phase; that belongs to
Phase 4.
