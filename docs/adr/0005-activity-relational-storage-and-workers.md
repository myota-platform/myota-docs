# ADR-0005: Relational activity storage and bounded workers

## Status

Accepted

## Decision

`myota-activity-service` owns normalized PostgreSQL tables for activations,
QSOs, worked-station facts, award definitions and versioned progress,
imports, corrections, jobs, statistics, notifications, requests and
issuances. It does not use the generic JSONB `service_state` table in durable
mode.

The service keeps one API boundary for activity and awards on port 8004. The
API is stateless and horizontally scalable. A bounded psycopg pool limits
database concurrency per pod. A PostgreSQL `COPY` staging path handles QSO
batches and ADIF worker output; unique deduplication keys make retries safe.

Background jobs handle ADIF parsing, award recalculation by historical rule
version, certificate rendering, statistics, notification delivery and
retries. NATS/outbox events remain the cross-service integration mechanism.

## Migration ownership

The activity repository contains the canonical migration. `myota-deploy`
applies a reviewed deployment copy during the shared migration job. This is a
deliberate compromise: the owning service reviews its schema, while one
deployment repository controls ordering, credentials, rollback guidance and
cluster execution.

## Consequences

- Participant progress and leaderboards read aggregate rows rather than
  scanning millions of QSOs.
- Historical awards remain reproducible because progress records include the
  award definition version.
- API pods can be scaled independently from import/render/notification workers.
- PostgreSQL remains the transactional source of truth; SeaweedFS/S3 stores only
  binary uploads and generated documents.
- The old JSONB state projection is not a migration target for activity data;
  existing prototype data must be imported through an explicit reconciliation
  job if retained.
