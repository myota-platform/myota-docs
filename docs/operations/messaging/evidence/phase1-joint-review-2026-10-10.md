# Phase 1 joint review: schemas, credentials, capacity, and recovery

**Review date:** 10 October 2026  
**Decision authority:** workspace owner delegated the review to the two-person project team (Volker Kerkhoff and Codex).  
**Status:** decisions recorded; implementation and production qualification remain incomplete.

This review covers every registered domain-fact payload schema (68), the ten selected work-command contracts, the deployed single-server NATS setup, and the relay/provisioner ownership. It is based on source and read-only host-cluster evidence. The schema registry and producer code are authoritative in `myota-contracts` and the owning service repositories; `myota-platform/contracts/` and the deployment files copied into `myota-platform` are synchronized mirrors. Deployment topology is owned by `myota-deploy`.

## Review decisions

### Payload schemas and data handling

**Decision — conditionally accept all 68 current fact schemas as source-derived inventory contracts, not as producer enforcement gates.** Each row below records the classification and current disposition. The immutable envelope and event identity are in scope for validation; schema acceptance does not approve unrestricted publication of every field. The source schemas intentionally remain additive and some dynamic objects remain open, so a producer validator must not be enabled until the required payload projection and compatibility tests are in place.

- Never publish passwords, password hashes, bearer/access tokens, NATS credentials, signing material, object bytes, or unbounded source documents. `identity.service-token.issued.v1` contains service and scopes only; the returned access token is not included (`myota-identity-service/identity.py`).
- Keep personally identifying and security data purpose-limited. `identity.login.failed.v1` currently contains email and remote address; `identity.login.succeeded.v1` contains account ID and remote address (`myota-identity-service/identity.py`). These facts have no selected business consumer. Restrict access to an approved security-audit group, retain only the event stream window, and use a follow-up major event version if fields are removed or transformed.
- For Activity/QSO and award facts, prefer stable resource IDs and bounded state summaries. Do not distribute QSO collections, award `facts`, certificate artifacts, or nested asset content unless a registered consumer needs those fields. Keep binary artifacts in object storage and use object identifiers.
- For Geodata facts, stable entity/import IDs and bounded lifecycle summaries are preferred. The current `geodata.import.preprocessed.v1` producer passes its whole result object, including `_records` and `_status`; `_records` can contain imported source features (`myota-geodata-service/geodata.py`, around the `store.event` call for this fact). Keep its current schema as an accurate record of v1, but do not enforce or expand v1. Before the producer is switched to the selected event topology, publish a compact versioned summary with counts and an import-run reference; keep feature records in PostGIS/object storage. A changed event shape requires a new event major version or an explicitly reviewed migration.
- Dynamic Programme rules/theme/OIDC/content/policy values and nested award templates remain open only where source code accepts them. Treat them as owner-controlled configuration, not trusted executable content; consumers must validate the subset they use.
- The 30-day event max age is a delivery/recovery window, not an archive or a grant to persist every sensitive field. PostgreSQL and service-owned audit state remain authoritative.

### Schema-by-schema disposition

The following rows were read from `myota-contracts/contracts/event-registry.json`; `payloadEvidence` and each schema file are generated/checked by the contracts workflow. “Conditionally accepted” means the schema accurately records current source shape and classification while the rules above gate producer enforcement.

| Event type | Owner | Data classification | Current subscriber disposition | Review decision |
|---|---|---|---|---|
| `activity.activation.closed.v1` | `activity-service` | `personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `activity.activation.created.v1` | `activity-service` | `personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `activity.adif.queued.v1` | `activity-service` | `personal-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `activity.entity.cascade-deleted.v1` | `activity-service` | `personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `activity.qso.recorded.v1` | `activity-service` | `personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `awards.definition.published.v1` | `activity-service` | `internal-configuration-and-personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `awards.definition.saved.v1` | `activity-service` | `internal-configuration-and-personal` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `awards.issued.v1` | `activity-service` | `personal-and-certificate-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `awards.rendered.v1` | `activity-service` | `personal-and-certificate-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `awards.request.created.v1` | `activity-service` | `personal-and-certificate-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.conflation.resolved.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity-deletion-job.completed.v1` | `geodata-service` | `personal-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity-deletion-job.created.v1` | `geodata-service` | `personal-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity-deletion-job.failed.v1` | `geodata-service` | `personal-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.candidate.created.v1` | `geodata-service` | `geospatial-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.deleted.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.entity-type-changed.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.geometry-type-changed.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.geometry-updated.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.import-approved.v1` | `geodata-service` | `geospatial-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.import-updated.v1` | `geodata-service` | `geospatial-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.location-enriched.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.location-updated.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.name-changed.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.reviewed.v1` | `geodata-service` | `geospatial-and-review-metadata` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.source-disappeared.v1` | `geodata-service` | `geospatial-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.entity.status-changed.v1` | `geodata-service` | `geospatial-and-review-metadata` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.cancellation-requested.v1` | `geodata-service` | `internal-operational-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.cancelled.v1` | `geodata-service` | `internal-operational-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.candidates.rejected.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.candidates.validated.v1` | `geodata-service` | `geospatial-and-review-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.failed.v1` | `geodata-service` | `internal-operational-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.preprocessed.v1` | `geodata-service` | `geospatial-and-source-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.processed.v1` | `geodata-service` | `internal-operational-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.import.processing.completed.v1` | `geodata-service` | `geospatial-and-operational-data` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.loadtest.cleaned.v1` | `geodata-service` | `internal-operational-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `geodata.refresh-schedule.created.v1` | `geodata-service` | `internal-configuration-and-source-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.account.admin-updated.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.account.created.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.account.deactivated.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.bootstrap-admin.created.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.callsign.added.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.callsign.evidence-submitted.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.callsign.primary-changed.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.callsign.retired.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.callsign.verified.v1` | `identity-service` | `personal` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.login.failed.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.login.succeeded.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.oidc.mapping.updated.v1` | `identity-service` | `internal-configuration` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.recovery.completed.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.recovery.requested.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.role-definition.created.v1` | `identity-service` | `internal-authorization` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.role-definition.updated.v1` | `identity-service` | `internal-authorization` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.role.assigned.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.roles.replaced.v1` | `identity-service` | `personal-and-security` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `identity.service-token.issued.v1` | `identity-service` | `security-sensitive` | activity-notifications-v1 | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.archived.v1` | `programme-service` | `internal-configuration` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.content.published.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.content.reviewed.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.content.saved.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.content.submitted.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.created.v1` | `programme-service` | `internal-configuration` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.entity-type.assigned.v1` | `programme-service` | `internal-configuration` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.entity-type.catalog-saved.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.entity-type.unassigned.v1` | `programme-service` | `internal-configuration` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.policy-draft.published.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.policy-draft.saved.v1` | `programme-service` | `internal-configuration-and-user-metadata` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |
| `programme.updated.v1` | `programme-service` | `internal-configuration` | none selected; no no-op durable | Conditionally accepted; apply the payload rules above before enforcement. |

The ten proposed work-command schemas are listed in the same registry under `workCommands`. Their payload evidence remains pending until each producer’s idempotency key, transaction coupling, recovery source, and maximum payload are asserted. Commands carry identifiers and small routing metadata, never source documents; the owning database remains the re-drive source. The six Activity job types remain distinct from facts; `NOTIFICATION_SEND` stays excluded because it represents state rather than an external delivery command.

**Open schema implementation gate:** add schema validation and focused fixtures for the exact producer projection, reject prohibited fields and oversized payloads, and cover all accepted consumer needs. In particular, complete the Geodata preprocessed projection before schema enforcement. Operations has no event-producing call site in the Phase 0 source audit; no Operations fact schema is required unless it becomes a producer.

### Credentials and permissions

**Decision — require distinct NKey credentials per runtime role, loaded from read-only Kubernetes Secret files backed by an operator-managed secret source.** Do not check seeds into Git, put them in Helm values, or share one application credential. Configure NATS authentication and TLS before cutover; restrict the ClusterIP through NetworkPolicy when the cluster CNI supports it.

| Role | Allowed purpose |
|---|---|
| `core-outbox` | Publish only registered fact subjects owned by Identity/Programme; no subscriptions. |
| `activity-outbox` | Publish only Activity fact subjects and, after cutover, Activity work subjects; no subscriptions. |
| `geo-outbox` | Publish only Geodata fact/work subjects; no subscriptions. |
| Activity notification consumer | Pull only its named fact durable and acknowledge only delivered messages. |
| Geodata work consumer | Pull only its registered work durables and acknowledge only delivered messages. |
| Deployment provisioner | Create/inspect only the named streams and durables; separate from runtime identities and unavailable to application pods. |
| Operations observer | Read stream/consumer metadata only; no message payload, consume, ACK, create/update/delete, or purge rights. |

JetStream management requests travel over NATS subjects, so the ACL must be tested against the actual NATS Python client’s API and ACK subjects; a broad `$JS.API.>` grant is limited to the one-shot provisioner. Keep clients inside the cluster and use TLS because connection authentication alone does not protect message contents in transit. Operations remains read-only as defined in `myota-operations-service` and `myota-docs/docs/operations/messaging/jetstream-admin-status.md`.

**Observed state:** `myota-deploy/deploy/helm/myota/templates/messaging.yaml` starts NATS without an auth config; the deployed chart exposes internal port 4222 and no credentials are mounted. This is an unauthenticated trusted-cluster boundary, not least-privilege authorization. Existing clients and provisioner therefore cannot be switched to auth until a secret source, server config, role ACLs, TLS files, and client wiring pass an isolated end-to-end test.

### Production capacity limits

**Decision — use a 5 GiB aggregate stream-byte budget on the current 8 GiB NATS PVC and reserve at least 3 GiB for filesystem headroom, metadata, compaction, and maintenance.** This is an initial proposal for controlled rollout, not a validated full-month capacity commitment. Do not set it as a live value until a representative 30-day profile and recovery test exist.

| Target stream | Initial `MaxBytes` proposal | `MaxMsgs` guard | `MaxAge` | `MaxMsgSize` |
|---|---:|---:|---:|---:|
| `MYOTA_EVENTS` | 1 GiB | 500,000 | 30 days | 1 MiB |
| `MYOTA_ACTIVITY_WORK` | 1 GiB | 250,000 | 30 days | 1 MiB |
| `MYOTA_GEODATA_WORK` | 3 GiB | 500,000 | 30 days | 1 MiB |

All streams use file storage, one replica on the current one-node cluster, `DiscardNew`, and finite byte, message, age, and per-message limits. A publish rejected at capacity remains pending in the owning outbox and follows its bounded retry/dead-letter path; no oldest message is silently discarded. Work past the 30-day window must be reconciled/redriven from its database using the same stable work ID before expiry; this recovery path must be qualified before activation.

**Evidence and limit:** a read-only query in the deployed outbox pods on 10 October 2026 found no unpublished rows. Across the 30-day query predicate, source rows actually span only 2–9 October in core (162 rows, max 387 bytes), 6–8 October in Activity (3,016 rows, max 150 bytes), and 6–9 October in Geodata (15,925 rows, max 373,607 bytes). The NATS PVC requests 8 GiB. This is less than an eight-day sample, includes load-test traffic in Geodata, and measures PostgreSQL outbox JSON rather than serialized NATS envelopes or the future command mix. The caps therefore remain a cautious starting budget. Recompute from a complete representative window, normalize test traffic, include envelope/index overhead and outage backlog, and lower/increase only while preserving the 3 GiB reserve.

### Restore, replay, and redrive

**Decision — PostgreSQL is the recovery authority; JetStream snapshots provide bounded transport recovery.** Back up each stream and its durable state to storage outside the NATS node, and retain the declarative topology/credential configuration separately. A recovery must restore into an isolated broker, recreate/validate durables from the checked-in provisioner, compare stream sequence and message IDs with source outboxes, verify consumer checkpoints and application idempotency, then authorize any production cutover. A PVC snapshot alone is not a qualified off-node backup.

- Replay a domain fact through a new isolated durable from an explicit sequence/time, validate it without production side effects, then discard that test durable. Do not rewind a production side-effecting consumer.
- Re-drive work from the owning database using the original work ID after checking its durable status and idempotency checkpoint. Never reconstruct work from stream contents alone.
- Preserve the existing Interest-retained `MYOTA_EVENTS` stream during transition. Under Interest retention, acknowledged messages may already be deleted; neither a new Limits policy nor a snapshot can recover records already removed. The currently observed stream was empty at the preceding read-only inspection, but this must be rechecked at cutover.
- Qualify PVC loss, off-node stream restore, durable recreation, duplicate delivery, disk-capacity rejection, relay retry/dead-letter, and bounded replay in a disposable namespace. Production restore/replay qualification remains open.

### Relay-side provisioning ownership

**Decision — `myota-deploy` owns one declarative, create-only provisioner; the three database-specific relays must not create streams/durables or alter retention.** It validates the complete topology and fails closed on drift. Provisioning uses a separate credential and runs before any relay can publish to new subjects. A relay may verify readiness but has no provisioning permissions.

Current `myota-deploy/services/outbox_worker.py` still creates/updates `MYOTA_EVENTS`, creates the legacy Activity/Geodata durables, and changes retention from Limits to Interest. The target provisioner in `myota-deploy/services/provision_jetstream.py` is create-only and validates drift, but the selected target limits are unset and the Kubernetes chart does not yet run it. Keep both facts explicit: do not run the target provisioner against the mixed legacy stream, and do not remove relay mutation until a migration-safe, deployment-owned preflight has created/validated the legacy compatibility topology and the current consumers can connect. The production transition must then use a stopped-relay inspection/snapshot barrier, provision the disjoint work streams, and switch one owner at a time without dual-publishing.

## Evidence and Phase 1 gates

| Review area | Decision complete | Implementation/qualification still required |
|---|---|---|
| 68 domain-fact schemas | Yes; per-event classification and consumer disposition are enumerated above. | Enforce only after projection, compatibility fixtures, payload-size checks, and the Geodata v1/v2 decision are implemented. |
| Ten work contracts | Design policy selected. | Complete each producer’s transaction/idempotency/recovery evidence and payload-size test. |
| Credentials | Role model and permission boundaries selected. | Supply secret material out of band; implement NATS auth/TLS, mount files, and prove allow/deny behavior per client role. |
| Capacity | Initial finite limits proposed against the 8 GiB PVC. | Full representative baseline, serialized NATS sizing, alerts, pressure/recovery test, and operator approval of final values. |
| Restore/replay | Recovery policy selected. | Create off-node backup and complete isolated loss/restore/replay qualification. |
| Relay provisioning | Single deployment owner and create-only model selected. | Add chart-run provisioning/readiness, remove mutation from relays at safe compatibility cutover, and prove no publish precedes preflight. |

### Exact source evidence

- Registry and all event schemas: `myota-contracts/contracts/event-registry.json`, `myota-contracts/contracts/schemas/events/`, `myota-contracts/contracts/schemas/event-envelope.schema.json`, `myota-contracts/contracts/events.md`.
- Payload source: `myota-identity-service/identity.py`; `myota-programme-service/programmes.py`; `myota-activity-service/activity.py`, `activity_repository.py`, `awards.py`; `myota-geodata-service/geodata.py`, `common.py`, `relational_state.py`.
- Current relay and selected topology: `myota-deploy/services/outbox_worker.py`, `services/jetstream_topology.py`, `services/provision_jetstream.py`; Helm source `deploy/helm/myota/templates/messaging.yaml`, `templates/deployment.yaml`, and `values.yaml`.
- Mirror evidence: matching deployment files in `myota-platform/deploy/`; matching contracts files in `myota-platform/contracts/`.
- Deployed evidence: K3s `default` context, node `spainip-k3s` Ready, Helm release `myota` revision 157, NATS single replica and 8 GiB PVC. All cluster queries and database aggregates in this record were read-only.

Phase 1 is not complete. The remaining exit criteria are checked implementation of the versioned envelope/subject registry, least-privilege authenticated provisioning, measured final limits, production-like backup/restore/replay evidence, and deterministic readiness before publishing. No phase after Phase 1 is claimed complete.


## Primary NATS references

- [NATS authorization](https://docs.nats.io/learn/security/authorization) describes publish/subscribe permissions as subject allow-lists and notes that JetStream API requests also require suitable permissions. This is why each client role needs an isolated allow/deny test against its actual APIs.
- [JetStream concepts](https://docs.nats.io/concepts/jetstream) describes stream and consumer persistence and replay. Replay is per consumer, so validation uses an isolated durable.
- [NATS TLS and authentication](https://docs.nats.io/learn/resilient-clients/tls-and-auth) covers client connection security and supported credentials.
- [JetStream stream API](https://docs.nats.io/reference/jetstream-api/stream) documents snapshot and restore operations. The planned drill includes consumer state verification and application database comparison because a broker snapshot does not make PostgreSQL state or the whole service system recoverable.
