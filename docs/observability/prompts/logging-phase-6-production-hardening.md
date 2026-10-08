# Phase 6 prompt - Production hardening and verification

Use this prompt in ChatGPT Work or Codex to implement Phase 6 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Perform the production-hardening and acceptance phase for MyOTA structured
logging.

Repositories to inspect:
- myota-platform/myota-deploy
- myota-platform/myota-docs
- all service repositories changed by Phases 1, 3 and 4

Requirements:
1. Validate the Loki label set is low-cardinality. Explicitly reject request IDs,
   trace IDs, correlation IDs, callsigns, entity IDs, QSO IDs and arbitrary URLs
   as labels.
2. Confirm production DEBUG logging is disabled by default.
3. Verify redaction for tokens, Authorization, passwords, DSNs, API/S3 keys,
   signing keys, ADIF contents and geodata/request payloads.
4. Confirm bounded Loki retention and persistent storage. Start from 14 days and
   document how to adjust it based on measured ingestion.
5. Add only log-specific exceptional-event alerts. Do not duplicate Prometheus
   ownership of availability, latency, error-rate or queue-depth conditions.
6. Candidate exceptional events include migration failure, permanent retention
   failure, JetStream max-delivery exhaustion, award rendering failure and
   unexpected authentication/signing failures. Implement only events that the
   current code can identify reliably.
7. Add operational queries/runbooks for:
   service ERROR investigation;
   correlation_id workflow reconstruction;
   job retry/failure investigation;
   trace-to-log and log-to-trace navigation;
   Loki/collector ingestion failure.
8. Verify Compose and Helm parity.
9. Execute an end-to-end acceptance scenario:
   HTTP request -> server span/log -> asynchronous message -> worker span/log,
   demonstrating stable correlation_id and usable trace/log links.
10. Record storage/ingestion observations and any capacity threshold that should
    trigger a future move from single-binary Loki or local PVC storage.
11. Update the implementation-plan status and parent observability documentation
    to reflect what is actually implemented, not planned.
12. Run relevant test suites, Helm lint/render and deployment checks and report
    objective evidence.

Do not claim completion for tests or runtime verification that were not actually
executed. Leave explicit follow-up items for any unverified production behavior.
