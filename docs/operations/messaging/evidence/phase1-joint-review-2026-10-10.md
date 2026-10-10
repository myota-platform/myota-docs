# Phase 1 joint review: schemas, capacity, recovery, and relay ownership

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

The ten proposed work types share the generic work envelope at
myota-contracts/contracts/schemas/work-command.schema.json. The registry still
marks each payload as pending owner evidence, so this review deliberately does
not approve its producer payload for enforcement. Each command must use bounded
identifiers and routing metadata, never source documents; the owning database
remains the re-drive source. The six Activity jobs remain distinct from facts;
NOTIFICATION_SEND stays excluded because it represents state rather than an
external delivery command.

| Work type | Target stream | Durable group | Current payload evidence | Review decision |
|---|---|---|---|---|
| `activity.qso-ingestion.v1` | `MYOTA_ACTIVITY_WORK` | `activity-qso-ingestion-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `activity.adif-import.v1` | `MYOTA_ACTIVITY_WORK` | `activity-adif-import-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `activity.award-recalculate.v1` | `MYOTA_ACTIVITY_WORK` | `activity-award-recalculate-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `activity.award-evaluation.v1` | `MYOTA_ACTIVITY_WORK` | `activity-award-evaluation-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `activity.pdf-render.v1` | `MYOTA_ACTIVITY_WORK` | `activity-pdf-render-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `activity.statistics-rebuild.v1` | `MYOTA_ACTIVITY_WORK` | `activity-statistics-rebuild-v1` | `pending-activity-idempotency-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `geodata.import-preprocess.v1` | `MYOTA_GEODATA_WORK` | `geodata-preprocessing-v1` | `pending-geodata-recovery-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `geodata.import-promotion.v1` | `MYOTA_GEODATA_WORK` | `geodata-import-promotion-v1` | `pending-geodata-recovery-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `geodata.entity-delete.v1` | `MYOTA_GEODATA_WORK` | `geodata-entity-deletion-v1` | `pending-geodata-recovery-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |
| `geodata.location-enrichment.v1` | `MYOTA_GEODATA_WORK` | `geodata-location-enrichment-v1` | `pending-geodata-recovery-and-payload-review` | Not approved for producer enforcement: confirm transaction coupling, idempotency/checkpoint, retry source, exact identifiers, and payload bound. |

**Open schema implementation gate:** before schema enforcement:

- Add validation and focused fixtures for each exact producer projection.
- Reject prohibited fields and oversized payloads.
- Cover all accepted consumer needs.
- Complete the Geodata preprocessed projection.

The Phase 0 source audit found no Operations event-producing call site. No
Operations fact schema is needed unless Operations becomes a producer.

### Cluster network boundary and Operations permissions

**Decision — keep NATS cluster-internal without authentication or TLS while it is
exposed only through the Kubernetes ClusterIP service.** The workspace owner
superseded the earlier NKey/TLS proposal. This decision means:

- Any pod that can reach the service is trusted as a NATS client.
- Do not expose the NATS client port through Ingress, NodePort, LoadBalancer,
  host port, or an external service.
- Revisit the decision before external exposure or admitting workloads that are
  not trusted at the cluster boundary.

**Observed state:** the deployed `myota-nats` service is `ClusterIP` on port
4222. The `myota` namespace has no NetworkPolicy, so the internal service is not
restricted to selected pods. The broker has no authentication configuration or
credentials, consistent with this decision. No runtime change is needed to
apply it.

Operations remains read-only by application behavior: inspect stream and consumer metadata only; do not publish, consume, acknowledge, purge, or change topology. This is an application boundary, not a NATS credential/ACL guarantee under the accepted unauthenticated cluster trust model.

### Production capacity limits

**Decision — use an initial 5 GiB aggregate stream-byte budget on the current
8 GiB NATS PVC, reserving at least 3 GiB for filesystem headroom, metadata,
compaction, and maintenance.** This is a controlled-rollout proposal, not a
validated full-month capacity commitment. Do not apply it live until a
representative 30-day profile and recovery test exist.

| Target stream | Initial `MaxBytes` proposal | `MaxMsgs` guard | `MaxAge` | `MaxMsgSize` |
|---|---:|---:|---:|---:|
| `MYOTA_EVENTS` | 1 GiB | 500,000 | 30 days | 1 MiB |
| `MYOTA_ACTIVITY_WORK` | 1 GiB | 250,000 | 30 days | 1 MiB |
| `MYOTA_GEODATA_WORK` | 3 GiB | 500,000 | 30 days | 1 MiB |

All streams use file storage, one replica on the current one-node cluster,
`DiscardNew`, and finite byte, message, age, and per-message limits.

- A capacity-rejected publish remains pending in its owning outbox and follows
  its bounded retry/dead-letter path. The broker does not silently discard the
  oldest message.
- Reconcile or redrive work from its database with the same stable work ID
  before the 30-day window expires. Qualify this recovery path before activation.

**Evidence and limit:** a read-only query in the deployed outbox pods on
10 October 2026 found no unpublished rows. Although the query covered 30 days,
the available source rows span less than eight days:

- Core: 2–9 October, 162 rows, maximum 387 bytes.
- Activity: 6–8 October, 3,016 rows, maximum 150 bytes.
- Geodata: 6–9 October, 15,925 rows, maximum 373,607 bytes.

The NATS PVC requests 8 GiB. The sample includes Geodata load-test traffic and
measures PostgreSQL outbox JSON, not serialized NATS envelopes or the future
command mix. Treat the caps as a cautious starting budget. Recompute from a
complete representative window, normalize test traffic, include envelope/index
overhead and outage backlog, and adjust only while preserving the 3 GiB reserve.

### Restore, replay, and redrive

**Decision — PostgreSQL is the recovery authority; JetStream snapshots provide
bounded transport recovery.** Keep declarative topology and credential
configuration separately from stream data. A qualified restore must:

1. Restore stream data and durable state into an isolated broker.
2. Recreate or validate durables with the checked-in provisioner.
3. Compare stream sequences and message IDs with source outboxes.
4. Verify consumer checkpoints and application idempotency.
5. Require explicit authorization before any production cutover.

A PVC snapshot alone is not a qualified off-node backup.

- Replay a domain fact through a new isolated durable from an explicit sequence/time, validate it without production side effects, then discard that test durable. Do not rewind a production side-effecting consumer.
- Re-drive work from the owning database using the original work ID after checking its durable status and idempotency checkpoint. Never reconstruct work from stream contents alone.
- Preserve the existing Interest-retained `MYOTA_EVENTS` stream during transition. Under Interest retention, acknowledged messages may already be deleted; neither a new Limits policy nor a snapshot can recover records already removed. The currently observed stream was empty at the preceding read-only inspection, but this must be rechecked at cutover.
- Qualify PVC loss, off-node stream restore, durable recreation, duplicate delivery, disk-capacity rejection, relay retry/dead-letter, and bounded replay in a disposable namespace. Production restore/replay qualification remains open.

### Relay-side provisioning ownership

**Decision — `myota-deploy` owns one declarative, create-only provisioner.** The
three database-specific relays must not create streams or durables, or alter
retention. The provisioner validates the complete topology and fails closed on
drift. It runs before any relay can publish to new subjects, using a separate
credential. A relay may verify readiness but has no provisioning permissions.

**Current implementation:** `myota-deploy/services/outbox_worker.py` still
creates or updates `MYOTA_EVENTS`, creates the legacy Activity/Geodata durables,
and changes retention from Limits to Interest. The create-only
`myota-deploy/services/provision_jetstream.py` validates drift, but selected
target limits are unset and the Kubernetes chart does not yet run it.

**Migration gate:** do not run the target provisioner against the mixed legacy
stream. Do not remove relay mutation until a migration-safe,
deployment-owned preflight has created and validated the legacy compatibility
topology and current consumers can connect. The production transition must use
this order:

1. Stop relays and inspect/snapshot the existing stream.
2. Provision the disjoint work streams.
3. Switch one owner at a time without dual-publishing.

## Evidence and Phase 1 gates

| Review area | Decision complete | Implementation/qualification still required |
|---|---|---|
| 68 domain-fact schemas | Yes; per-event classification and consumer disposition are enumerated above. | Enforce only after projection, compatibility fixtures, payload-size checks, and the Geodata v1/v2 decision are implemented. |
| Ten work contracts | Design policy selected. | Complete each producer’s transaction/idempotency/recovery evidence and payload-size test. |
| Cluster network boundary | Accepted: no NATS auth/TLS while the broker remains ClusterIP-only and the cluster workload boundary is trusted. | Keep the service internal; re-evaluate before external exposure or admitting untrusted workloads. The current namespace has no NetworkPolicy. |
| Capacity | Initial finite limits proposed against the 8 GiB PVC. | Full representative baseline, serialized NATS sizing, alerts, pressure/recovery test, and operator approval of final values. |
| Restore/replay | Recovery policy selected; basic isolated snapshot/restore and replay passed. | Off-node backup, PVC-loss recovery, database reconciliation, and production-like qualification remain open. |
| Relay provisioning | Single deployment owner and create-only model selected. | Add chart-run provisioning/readiness, remove mutation from relays at safe compatibility cutover, and prove no publish precedes preflight. |

### Exact source evidence

- Registry and all event schemas: `myota-contracts/contracts/event-registry.json`, `myota-contracts/contracts/schemas/events/`, `myota-contracts/contracts/schemas/event-envelope.schema.json`, `myota-contracts/contracts/events.md`.
- Payload source: `myota-identity-service/identity.py`; `myota-programme-service/programmes.py`; `myota-activity-service/activity.py`, `activity_repository.py`, `awards.py`; `myota-geodata-service/geodata.py`, `common.py`, `relational_state.py`.
- Current relay and selected topology: `myota-deploy/services/outbox_worker.py`, `services/jetstream_topology.py`, `services/provision_jetstream.py`; Helm source `deploy/helm/myota/templates/messaging.yaml`, `templates/deployment.yaml`, and `values.yaml`.
- Recovery procedure: `myota-docs/docs/operations/messaging/jetstream-recovery.md`.
- Mirror evidence: matching deployment files in `myota-platform/deploy/`; matching contracts files in `myota-platform/contracts/`.
- Deployed evidence: K3s `default` context, node `spainip-k3s` Ready, Helm release `myota` revision 157, NATS single replica and 8 GiB PVC. All cluster queries and database aggregates in this record were read-only.

At the time this joint-review snapshot was recorded, Phase 1 was not complete.
The contracts-owned v1 envelope schemas and registry
for 68 facts and ten selected work commands are now verified by focused tests
and Contracts CI run
[38045763460](https://github.com/myota-platform/myota-contracts/actions/runs/38046227981).
The deploy-owned create-only topology definition and drift checks also pass
focused and isolated broker checks. This completes the contract/subject-registry
exit criterion; it does not mean that the current relay enforces the new
envelope or subject rules. At that review point, remaining exit criteria were deterministic provisioning
with a readiness barrier,
payload projection and unknown-route enforcement, measured final limits, and
qualified off-node backup/restore/replay. NATS authentication and TLS are not
requirements while the broker remains cluster-internal under the accepted trust boundary. The
[recovery runbook](../jetstream-recovery.md) now records the procedure, but
qualification evidence remains open. No phase after Phase 1 is claimed complete.


## Primary NATS references

- [JetStream concepts](https://docs.nats.io/concepts/jetstream) describes stream and consumer persistence and replay. Replay is per consumer, so validation uses an isolated durable.
- [JetStream stream API](https://docs.nats.io/reference/jetstream-api/stream) documents snapshot and restore operations. The planned drill includes consumer state verification and application database comparison because a broker snapshot does not make PostgreSQL state or the whole service system recoverable.


## 10 October 2026 — disposable restore and replay drill

A disposable namespace on the host K3s cluster ran two clean NATS 2.10
JetStream servers. The NATS CLI v0.2.3 created a finite Limits stream, published
three synthetic messages, and created a pull durable. After acknowledging
sequence 1 and leaving sequences 2–3 pending, a stream backup was restored to
the second server. The restored stream contained all three messages and the
durable preserved its acknowledged position with two unprocessed messages.
The durable then consumed and acknowledged those two messages. A new isolated
durable with DeliverPolicy.ALL replayed all three restored messages, including
sequence 1. The restored stream's message 1 was also read directly by sequence.
The test therefore verified the basic snapshot data, consumer-state restore,
resume, and independent bounded replay behavior.

The test created no application/database data and did not connect to the
production NATS service. The temporary namespace was deleted and verified
absent; the CLI binary and snapshot were removed from /tmp. This was a same-host
isolated drill, not an off-node backup, PVC-loss test, database reconciliation,
authenticated-ACL test, or production recovery qualification.

## Phase 1 closeout decisions — 10 October 2026

The project team is Volker Kerkhoff and Codex. At the workspace owner's
direction, off-node recovery is deferred for the current single-node scope.
The accepted residual risk is loss of the bounded JetStream transport window
with node/cluster loss; PostgreSQL remains the authority for business state,
reconciliation, and work redrive.

Volker accepts the conservative initial caps (1 GiB facts, 1 GiB Activity work,
3 GiB Geodata work; finite message/age/payload limits and 3 GiB PVC reserve)
despite the available database sample covering fewer than ten days and
including load-test traffic. Treat those caps as fixed ceilings for the initial
single-node scope. Do not increase them without representative 30-day
serialized-traffic and outage-backlog evidence.

The deploy repository now contains an optional fail-closed Helm pre-upgrade
provisioner hook. It is disabled by default and requires explicit confirmation
of the migration gate; the current mixed Interest-retained `MYOTA_EVENTS`
stream must still not be targeted. The isolated 10 October closeout also
verified a replacement local PVC, stream/durable restore, replay, idempotent
provisioning, drift rejection, and `DiscardNew` pressure behavior. It did not
touch the production namespace. Relay cutover, payload projection/enforcement,
production outbox watermark comparison, and retry/dead-letter qualification
remain Phase 2 gates. See the
[Phase 1 completion evidence](phase1-completion-2026-10-10.md).
