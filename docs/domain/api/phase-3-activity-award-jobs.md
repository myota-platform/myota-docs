# Phase 3 — Activity and award jobs

**Status:** complete  
**Completed:** 2026-10-11  
**Owner:** myota-activity-service; OpenAPI and client facades are owned by
myota-contracts.

Activity exposes durable asynchronous work through job resources. Domain state,
idempotency records, work claims, progress, and recovery remain Activity-owned.
The contract is maintained in
[canonical OpenAPI](https://github.com/myota-platform/myota-contracts/blob/main/contracts/openapi.yaml).

## Current operations

| Work | Current API |
| --- | --- |
| Activation updates | **PATCH /v1/activations/{activationId}** |
| QSO ingestion | **POST /v1/qso-ingestions**; read with **GET /v1/qso-ingestions/{ingestionId}** |
| QSO correction review | **POST /v1/qsos/{qsoId}/corrections/{correctionId}/review** |
| Statistics rebuild | **POST /v1/statistics/rebuild-jobs**; read the returned job resource. |
| Award evaluation | **POST /v1/awards/evaluation-jobs**; read the returned job resource. |
| Award recalculation | **POST /v1/awards/{awardId}/recalculation-jobs**; read the returned job resource. |
| Award issuance | **POST /v1/awards/requests/{requestId}/issuances** |
| Certificate rendering | **POST /v1/awards/issuances/{issuanceId}/render-jobs**; read the returned job resource. |
| Certificate artifact | **GET /v1/awards/issuances/{issuanceId}/artifact** |
| Award progress | **GET /v1/awards/progress?awardId=...&participantId=...** |

Job requests remain idempotent when supplied an Idempotency-Key. Workers
claim rows using bounded database locking, record attempts and errors, retry
transient failures with bounded backoff, and leave exhausted work inspectable
in a terminal failure state.

Audited commands such as submit, review, publish, retire, and explicit job
confirmation remain commands because each represents a domain decision or
side effect. Programme owns programme policy and configuration. Activity owns
award definitions, evaluation, progress, requests, rendering, notifications,
and permanent issuance records.

## Evidence

- Activity quality and image workflows pass on current main.
- The API inventory and generated client checks cover the current operation
  set.
- Existing authorization, idempotency, audit, retry, and worker recovery
  checks remain the behavioral boundary.

See the [Activity and award job diagram](../../architecture/diagrams/activity-award-jobs.md),
the [REST API consolidation record](rest-consolidation-plan.md), and the
[cross-repository phase evidence](../../architecture/evidence/review-phases-1-4-2026-10-11.md).
