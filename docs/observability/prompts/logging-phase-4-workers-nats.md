# Phase 4 prompt - Workers, NATS and asynchronous correlation

Use this prompt in ChatGPT Work or Codex to implement Phase 4 of the
[structured logging plan](../logging-implementation.md).

## Prompt

Add structured logging and distributed correlation to MyOTA asynchronous workers,
outbox publishers and NATS/JetStream consumers.

Inspect all current workers in the relevant service and deployment repositories,
including at least:
- activity-worker
- activity-notifications
- activity-adif-retention
- geodata-import-processing
- geodata-import-retention
- identity-maintenance
- outbox-core
- outbox-activity
- outbox-geo

Requirements:
1. Use the Phase 1 structured logging helpers.
2. Standardize lifecycle events:
   job.received, job.started, job.completed, job.retry, job.failed.
3. Include job.type/job.id where available, duration, retry/delivery attempt,
   sanitized error.type/error.message and correlation_id.
4. Instrument messaging with OpenTelemetry semantic concepts:
   messaging.system=nats, destination/subject, message/event ID,
   messaging.operation and delivery count where available.
5. Propagate trace context and correlation_id through NATS/JetStream message
   headers or the existing event envelope without breaking backwards
   compatibility.
6. Producers should inject context; consumers should extract it and create a
   consumer/process span. Preserve event_id independently of trace_id.
7. A business workflow may span several traces; correlation_id must remain stable
   through retries and subsequent jobs.
8. Do not put payloads, ADIF contents, geometry bodies, secrets or arbitrary user
   data in log records.
9. Avoid creating high-cardinality Loki labels; this phase emits structured
   fields only.
10. Add tests covering propagation, retry behavior, failed jobs, malformed/missing
    trace headers and backwards-compatible consumption of older messages.
11. Verify that dead/retried messages remain diagnosable and that logging failure
    does not change ACK/NAK business semantics.
12. Update relevant operations/event-contract documentation.

Report changed repositories, tests, example sanitized records and any message
contract changes.
