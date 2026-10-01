# Phase 3 activity and award jobs

Phase 3 adds resource-oriented asynchronous execution to the shared activity
and awards API on port 8004. Existing routes remain available as compatibility
aliases; the canonical operation IDs and schemas are in
[`myota-contracts/contracts/openapi.yaml`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).

## Preferred resources

| Resource | Preferred operation | Purpose |
| --- | --- | --- |
| Activation lifecycle | `PATCH /v1/activations/{activationId}` | Close an activation with `status=CLOSED`, preserving the evaluated programme-rule snapshot. |
| QSO ingestion | `POST /v1/qso-ingestions`; `GET /v1/qso-ingestions/{ingestionId}` | Queue high-volume normalized QSOs for PostgreSQL COPY-based insertion, deduplication, aggregate updates, and award recalculation. |
| Correction review | `POST /v1/qsos/{qsoId}/corrections/{correctionId}/review` | Keep correction review under the QSO resource while retaining the audit workflow. |
| Statistics rebuild | `POST /v1/statistics/rebuild-jobs`; `GET .../{jobId}` | Reproducibly rebuild programme, activator, hunter, entity, and jurisdiction aggregates. |
| Award evaluation | `POST /v1/awards/evaluation-jobs`; `GET .../{jobId}` | Evaluate a programme-owned rule tree against precomputed participant facts. |
| Award recalculation | `POST /v1/awards/{awardId}/recalculation-jobs`; `GET .../{jobId}` | Recalculate historical definitions using the award rule version captured in the job. |
| Award issuance | `POST /v1/awards/requests/{requestId}/issuances` | Create the permanent issuance record without changing issuance semantics. |
| Certificate rendering | `POST /v1/awards/issuances/{issuanceId}/render-jobs`; `GET .../{jobId}` | Queue retryable PDF generation from the configured background/signature assets. |
| Certificate artifact | `GET /v1/awards/issuances/{issuanceId}/artifact` | Retrieve the durable issued certificate through a short-lived object-storage URL. |
| Award progress | `GET /v1/awards/progress?awardId=...&participantId=...` | Read progress from aggregate facts; the legacy POST calculation remains an alias. |

## Durable job lifecycle

All asynchronous resources use the activity-owned `activity_job` table and the
same bounded worker pool. Jobs are idempotent where a request supplies an
`Idempotency-Key`, claim work with `FOR UPDATE SKIP LOCKED`, and expose attempts,
timestamps, status, and the last retry error. Transient failures are requeued
with bounded exponential backoff; jobs exceeding the configured attempt limit
become `FAILED` and remain inspectable.

```text
resource request
  -> QUEUED
  -> RUNNING (worker claim)
  -> SUCCEEDED
       or QUEUED again after transient failure
       or FAILED after the retry budget is exhausted
```

The worker handlers are explicit:

- `QSO_INGESTION` normalizes records, validates activation windows, inserts
  through the repository batch/COPY path, updates aggregate counters, and
  enqueues award recalculation;
- `ADIF_IMPORT` scans, parses, deduplicates, validates, and imports the stored
  object asynchronously;
- `STATISTICS_REBUILD` writes versioned reproducible snapshots;
- `AWARD_EVALUATION` and `AWARD_RECALCULATE` store versioned progress and emit
  qualification notifications;
- `PDF_RENDER` writes the immutable certificate artifact to SeaweedFS/S3;
- `NOTIFICATION_SEND` delivers queued account, import, award, and security
  notifications.

## Compatibility and security

The former close, batch, ADIF, correction-review, statistics-rebuild, award
evaluation/progress, award recalculation, issuance, and certificate-render
routes are retained as aliases. They emit `Deprecation: true` and
`Sunset: 2027-04-01T00:00:00Z`; preferred resource routes do not. The old
handlers delegate to the same repository and worker methods, so authorization,
idempotency, audit, deduplication, and retry behavior cannot diverge.

Award rules remain programme-owned. Recalculation captures the award version;
publishing a new definition does not silently rewrite historical issuance
records. New qualification and import outcomes are notification events, while
certificate issuance remains permanent and separately addressable from its
render job.

See the [activity and award job diagram](diagrams/activity-award-jobs.md) and
the [REST consolidation plan](api-rest-consolidation-plan.md).
