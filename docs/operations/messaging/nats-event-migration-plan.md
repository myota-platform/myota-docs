# NATS JetStream event and work-queue migration plan

**Status:** Phase 0 and Phase 1 contract/topology work are complete. Phase 1
exit criteria are met by checked-in contracts, deploy-owned create-only
provisioning and gated Helm preflight, and isolated single-node qualification.
No producer or consumer path has been cut over. Phase 2 compatibility and
production rollout gates remain open.

**Progress tracking:** leave items unchecked until evidence is available; mark
`[x]` only when the work is verified. A phase is complete only after all its
exit-criteria items are checked and the evidence is recorded.

**Scope:** all transactional outbox events and asynchronous consumers across
MyOTA services, including the remaining database-polled activity jobs where
they are the consumer side of accepted asynchronous work.

**Owner:** cross-repository change coordinated through `myota-docs`; code changes
belong in each owning service and deployment/contracts repository.

## Goal

Make NATS JetStream the durable, observable delivery layer for cross-service
domain events and the accepted asynchronous work commands selected in ADR-0008.
Keep scheduled maintenance and reconciliation on their scheduler/recovery paths.
Keep the transactional outbox as the atomic database-to-broker handoff: domain
mutation and outbox row must commit together, and a relay publishes with a stable
message ID. Consumers must be independently deployable, horizontally scalable
where appropriate, idempotent, and safe under at-least-once delivery.

This is a completion and standardization effort, not a greenfield adoption. The
repository already has transactional outboxes in Identity, Programme, Activity,
Geodata, and Operations; three database-specific relays publish to JetStream;
Geodata has four durable work consumers; and Activity has a durable pull
notification consumer. The remaining work is to prove complete event coverage,
close asynchronous-work paths that still poll PostgreSQL, establish consistent
contracts and lifecycle guarantees, and qualify rollout and recovery.

## Current inventory and boundaries

| Owner / database | Current outbox producers | Current broker publishing | Current consumers / remaining path |
|---|---|---|---|
| Identity / `myota_core` | Account, callsign, role, authentication/recovery, OIDC mapping, and service-token lifecycle events. Examples: `identity.account.created.v1`, `identity.callsign.verified.v1`, `identity.account.deactivated.v1`. | Published by the shared `core-outbox` relay. | No Identity-owned JetStream consumer found. Activity notifications observe `myota.events.>` and translate supported Identity events into notices. Verify every event is either intentionally consumed or documented as an integration event with no current subscriber. |
| Programme / `myota_core` | Programme create/update/archive, entity-type catalogue/assignment, content workflow, and policy-draft events. | Shared `core-outbox` relay. | No Programme-owned JetStream consumer found. Verify event subscribers and future ownership. |
| Activity / `myota_activity` | Activation/QSO lifecycle, ADIF queue, activity cascade deletion, award definition/request/issuance/rendering, and other activity/award events. | `activity-outbox` relay. | Activity notification consumer currently subscribes broadly. The selected target uses scoped filters for Identity facts and the two Geodata review/status facts it handles. Migrate six accepted jobs (QSO ingestion, ADIF import, award recalculation/evaluation, PDF rendering, and statistics rebuild) to Activity work subjects. Exclude the state-only `NOTIFICATION_SEND` job; correct or remove it, and create a separate provider-backed command if external delivery is added later. |
| Geodata / `myota_geo` | Import lifecycle, validation/promotion, entity review/change/deletion, location enrichment, cancellation, recovery, and operational events. | `geo-outbox` relay. Generic events use `myota.events.<event_type with dots replaced by underscores>`; work dispatch uses allowlisted `payload.natsSubject`. | `geodata_import_worker.py` has durable pull consumers: `geodata-preprocessing-v1`, `geodata-import-processing-v2`, `geodata-entity-deletion-v1`, and `geodata-location-enrichment-v1`, plus stale-cancellation and pending-deletion database reconcilers. Keep reconciliation as recovery, not a second normal work queue. |
| Operations / `myota_core` | The shared state adapter has an outbox/event helper, but the Phase 0 source scan found no Operations event-producing call sites; do not treat helper support as emitted events. | Shared `core-outbox` relay can read the core database outbox; no Operations-specific event stream was identified. | Operations reads JetStream stream/consumer status and stores samples; it is deliberately read-only and is not a business-event consumer. Preserve this boundary. |

The relay is currently shared code in `myota-deploy/services/outbox_worker.py`
and is deployed once per physical database (`core`, `activity`, `geo`). It
provisions one shared file-backed `MYOTA_EVENTS` stream with Interest retention
and known durable filters before publishing. The durable set currently includes
Activity notifications plus the four Geodata work filters. The selected target
is `MYOTA_EVENTS` with Limits retention for bounded domain-fact replay, plus
separate `MYOTA_ACTIVITY_WORK` and `MYOTA_GEODATA_WORK` streams with WorkQueue
retention for competing work. PostgreSQL domain state, outbox/dead-letter
records, and consumer idempotency/checkpoint state remain authoritative.

### Event/work inventory evidence

The [Phase 0 inventory and selected topology](nats-event-migration-inventory.md)
records the repository-source event scan, current subject derivation, explicit
Geodata work routes, Activity job queue, recovery migration, relay and consumer
behavior, deployment ownership, and mirror comparisons. It distinguishes
observed subscribers from proposed consumers and records unresolved evidence.
The selected target is documented in
[ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
The registry at `myota-contracts/contracts/event-registry.json` is the
machine-checked catalogue for 68 fact types and ten work commands. Phase 1
contract/topology and isolated single-node qualification are complete. Runtime
enforcement, compatibility rollout, and production cutover remain open.

Do not silently turn every domain event into a command queue. Preserve the
distinction:

- **Domain event:** a committed fact, delivered to each interested durable
  consumer. Use an explicit event subject convention and consumer-specific
  durable filters.
- **Work command:** a durable request for one competing worker group. Use a
  dedicated subject and durable pull consumer; replicas in the same group share
  the durable, while independent work handlers use separate durables.
- **Operational inspection:** read-only broker metadata; it must not consume,
  acknowledge, purge, or change durable configuration.

## Target design and invariants

1. Every accepted asynchronous operation persists its domain state and a
   versioned outbox/work record in one transaction. No event depends on an API
   process-local list, executor, or best-effort publish. Scheduled maintenance
   and reconciliation remain owned by their scheduler/database recovery path.
2. The relay publishes the complete envelope (event ID/type/time, producer,
   aggregate identity, correlation/causation context, payload, and
   `envelopeVersion: 1`)
   using the stable event/work ID as JetStream message ID. It marks the outbox
   row published only after broker acknowledgement. Retry exhaustion is
   visible and recoverable from a dead-letter record.
3. Delivery is at least once. Every consumer has a stable durable identity,
   explicit ack, bounded redelivery, timeouts/concurrency, durable side-effect
   idempotency, and a documented poison-message path. Acknowledgement follows
   committed side effects/checkpoint state.
4. Queue workers use pull consumers and shared durable names for replicas.
   Domain event fan-out uses one durable per independent consumer group. Never
   share a durable between unrelated handlers or create per-replica durables.
5. Stream and consumer provisioning is controlled, drift-checked, and validated
   before producers use new subjects. The target topology is `MYOTA_EVENTS`
   (`myota.events.>`, Limits, file storage, 30-day max age) plus
   `MYOTA_ACTIVITY_WORK` and `MYOTA_GEODATA_WORK` (disjoint `myota.work.*`
   subject prefixes, WorkQueue, file storage). All streams use finite
   `MaxAge`, `MaxBytes`, `MaxMsgs`, and `MaxMsgSize` with `DiscardNew`. The
   accepted initial caps are 1/1/3 GiB, 500k/250k/500k messages, 30-day age,
   and 1 MiB maximum payload, with 3 GiB reserved on the current 8 GiB PVC;
   Volker accepts the short-sample caveat for the single-node scope. Keep one
   replica on today's single-server deployment; use three only after a
   three-server JetStream cluster is deployed. Off-node recovery is deferred.
   Do not raise limits without representative 30-day traffic and outage-backlog
   data.
6. JetStream is delivery infrastructure, not the permanent event archive.
   Choose retention according to the required processing and recovery window;
   keep domain history and outbox/dead-letter evidence in service-owned storage.
   Retained facts can be replayed through a new isolated durable with an
   explicit start position; work is redriven from its owning database with the
   same work ID. Acknowledged-event replay must use an explicit supported path.
   Local disposable-PVC restore/replay is qualified; node/cluster loss may
   discard the bounded transport window.
7. Preserve service ownership and database boundaries. Identity, Programme,
   and Operations share the core database today; do not interpret the shared
   relay as a shared domain repository. No service reads another service's
   outbox or consumer tables.
8. Keep `myota-platform` / `myota-deploy` mirrors in sync with owning service
   source. Contracts and subject/consumer inventories are published from the
   contracts/docs authority.
9. Operations visibility reports stream and durable state but does not mutate
   it. Add bounded-cardinality metrics for relay health/backlog, publish
   failures/dead letters, consumer pending/ack-pending/redeliveries/oldest age,
   handler outcomes, and end-to-end event age. Do not label metrics with event
   IDs, aggregate IDs, or other unbounded values.
10. Keep the consumer APIs/web clients out of direct NATS access. NATS is
    cluster-internal through a ClusterIP service; the accepted trust boundary
    assumes any pod able to reach that service is trusted. Do not expose NATS
    outside the cluster. Operations remains read-only for broker inspection.

### Selected topology — ADR-0008

The following target is selected for implementation. It does not describe the
current deployment, and its implementation and production qualification remain
open. See [ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md)
and the [event/work inventory](nats-event-migration-inventory.md) for rationale
and evidence.

| Stream | Subject capture | Retention and storage | Use |
|---|---|---|---|
| `MYOTA_EVENTS` | `myota.events.>` | Limits, file storage, 30-day max age, finite `MaxBytes`, `MaxMsgs`, and `MaxMsgSize`; `DiscardNew` | Committed domain facts, with one durable per intended independent consumer group and bounded replay from an isolated durable. |
| `MYOTA_ACTIVITY_WORK` | `myota.work.activity.>` | WorkQueue, file storage, finite age/byte/message/payload limits; `DiscardNew` | Six selected Activity jobs: QSO ingestion, ADIF import, award recalculation, award evaluation, PDF rendering, and statistics rebuild. One disjoint filter/durable per work kind; replicas share the durable. |
| `MYOTA_GEODATA_WORK` | `myota.work.geodata.>` | WorkQueue, file storage, finite age/byte/message/payload limits; `DiscardNew` | Geodata preprocessing, import promotion, confirmed deletion, and location enrichment. One disjoint filter/durable per work kind; replicas share the durable. |

Facts use `myota.events.<eventType>` with dotted tokens preserved. Keep
`envelopeVersion: 1` separate from the `.vN` event type version. Use checked-in
per-event JSON Schemas and the subject/consumer registry in `myota-contracts`.
Do not migrate the synthetic Activity `NOTIFICATION_SEND` state-only job; if
external delivery is introduced, define a separate provider-backed command.
Keep scheduled maintenance and database reconciliation as scheduler/recovery
paths. Keep Operations read-only for broker inspection.

Delivery is at least once. `Nats-Msg-Id` deduplication is a short-window
optimization; database idempotency and checkpoint state define correctness.
Persist an application dead-letter record before terminating terminal failures
and provide authorized, audited redrive. Redrive work from the owning database
with the same work ID. Replay facts with a separate durable and explicit start
position; never rewind a production side-effecting consumer. PostgreSQL remains
the source of truth, and JetStream is not a permanent event archive. Keep stream
replicas at one on today's single-server topology; configure three only after a
three-server JetStream cluster exists. Use one deployment-owned, drift-checked
provisioner rather than concurrent mutation from the three relay instances.

Rollout must account for the current mixed `MYOTA_EVENTS` stream. Stop relays and
inspect retained messages/consumer state before changing its retention from
Interest to Limits; this live change cannot restore messages already deleted.
Create the two WorkQueue streams on the disjoint `myota.work.*` subjects, provision
their durables, and route pending legacy Geodata work through one publish path.
Do not dual-publish. Drain old work and remove old filters/routes only after
rollback and database recovery checks pass.

## Phased implementation

### Phase 0 — Complete inventory and decision record

**Status: complete.** The authoritative repository inventory, selected target
topology, decision record, and evidence ownership are recorded in the
[Phase 0 inventory](nats-event-migration-inventory.md) and
[ADR-0008](../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md).
Deploy and platform copies are identified as synchronized mirrors.

- [x] Inventory authoritative producer, relay, consumer, worker, recovery, test,
      and deployment sources across all repositories named in the inventory.
- [x] Distinguish committed facts, competing work, scheduled/reconciliation paths,
      and synchronous operations; record source owners and current boundaries.
- [x] Select the proposed target stream/retention/subject design and record its
      tradeoffs without treating JetStream as an archive.
- [x] Record unresolved evidence, responsible repositories, current constraints,
      and gates for producer/consumer changes.

**Exit criteria**

- [x] The evidence-backed inventory covers the Phase 0 scope and cites exact paths.
- [x] The selected target design and operational boundaries are recorded in ADR-0008.
- [x] Remaining evidence gaps and their owners/gates are recorded; Phase 1 contract
      and topology work may proceed while runtime paths remain gated.

### Phase 1 — Contracts, topology, provisioning, and operational safety

**Work**

- [x] Define the immutable event envelope v1 and work-command envelope v1 in
      checked-in JSON Schemas. Require stable UUID identity, UTC timestamps,
      producer and aggregate identity, bounded correlation/causation fields,
      and strict outer-envelope fields; keep relay attempts outside the
      envelope and the envelope version independent from event type version.
      See `myota-contracts/contracts/schemas/event-envelope.schema.json`,
      `myota-contracts/contracts/schemas/work-command.schema.json`, and
      `myota-contracts/contracts/events.md`.
- [x] Record runtime envelope enforcement, payload projection, serialized
      size enforcement, trusted correlation/causation propagation, and unknown
      version handling as Phase 2 relay work. The current relay still emits its
      legacy envelope and subject mapping; Phase 1 deliberately does not change
      runtime paths.
- [x] Register all 68 domain facts and ten selected work commands with exact
      subjects, fact/work classification, source owner, schema path, and
      subscriber disposition. Keep work subjects disjoint in
      `myota.work.activity.*` and `myota.work.geodata.*`; map each command
      to its intended stream and unique durable. Contracts checks require
      event dispositions and checked-in schemas, compare every work stream,
      subject, and durable against the deploy-owned topology, and audit
      event-like source literals across the five services. The complete contracts workflow passed after these fixtures were added ([run 38045763460](https://github.com/myota-platform/myota-contracts/actions/runs/38046227981)).
- [x] Define the complete subject and version registry and test unknown
      entries at the contract boundary. Runtime rejection and retry/dead-letter
      persistence remain Phase 2 work because they require a compatible relay
      rollout.
- [x] Implement the deploy-owned, create-only provisioner with configuration
      drift checks and explicit validation of stream and pull-consumer
      correctness settings. The isolated broker check passed creation,
      idempotent rerun, and drift rejection; focused topology tests pass.
      Capacity has no defaults and requires explicit positive values.
- [x] Add the optional, fail-closed Helm pre-upgrade provisioner hook. Helm
      waits for create-only topology validation before updating workloads. It
      requires explicit migration-gate confirmation and is disabled by default.
      Removing stream/durable mutation from the three relays remains Phase 2
      work; never run the target provisioner against the current mixed stream.
- [x] Record the accepted NATS network boundary: keep the broker ClusterIP-only
      inside the cluster and rely on the trusted-cluster boundary without NATS
      authentication or TLS. The live `myota` namespace has no NetworkPolicy, so
      all pods with network reachability are trusted to connect. Revisit this
      decision before exposing NATS outside the cluster or admitting untrusted
      workloads.
- [x] Encode the selected target retention/storage policy in the declarative
      topology: finite Limits retention for facts, WorkQueue retention for
      Activity and Geodata work, file storage, one replica on the single-server
      cluster, DiscardNew, finite per-message size, and ten explicit work
      durable filters. The provisioner refuses to invent capacity defaults.
- [x] Select conservative finite starting caps of 1/1/3 GiB, 500k/250k/500k
      messages, 30 days, and 1 MiB maximum payload, reserving 3 GiB on the 8 GiB
      PVC. The measured database sample spans fewer than ten days and contains
      load-test traffic; Volker accepts this bounded evidence risk for the
      single-node scope. Keep these caps off the live mixed stream and do not
      raise them without representative 30-day traffic and outage-backlog data.
- [x] Document PostgreSQL as recovery authority and JetStream as a bounded
      delivery/replay window. Add the [recovery and replay runbook](jetstream-recovery.md)
      for isolated restore, fact replay, work redrive, legacy Interest-retention
      limitations, and sensitive-data handling.
- [x] Qualify isolated stream/durable creation, idempotent provisioner rerun,
      drift rejection, local disposable-PVC restore, bounded replay, and
      `DiscardNew` capacity rejection. Off-node recovery is deferred at the
      project owner's direction; node/cluster loss may discard the bounded
      transport window, so reconcile facts and redrive work from PostgreSQL.
      Production watermark comparison and relay retry/dead-letter qualification
      remain Phase 2 gates.
- [x] Add contract fixtures and CI checks for unique event/work subjects,
      per-event schemas, subscriber dispositions, source provenance, and
      registered work-to-durable topology. A new producer literal without a
      registry disposition fails the workspace source audit.

The machine-readable registry covers 68 domain facts and ten selected work
commands. The six current Geodata work/recovery event types map to four target
commands. Each work command has a bounded payload schema, source-linked producer
and consumer paths, and an owning-row `workId` source. These target contracts do
not authorize runtime publication. `myota-contracts/contracts/event-registry.json`
records exact producer source paths for all 68 facts. Its workspace audit checks those references
and classifies Python event-like source literals as a fact or mapped legacy work
type; the 10 October audit found no unclassified Python literals across the five
service repositories; the contracts CI audit passed on main. Payload shapes for
all 19 Identity, 12 Programme, 10 Activity, and 27 Geodata facts now have
source-derived payload schemas generated from their authoritative producer files.
Identity, Programme, Activity, and Geodata schema checks passed in Contracts CI
(run 38042564323 for Geodata; run 38040217352 for Activity). The delegated schema
review is complete; payload projection, prohibited-field/size checks, and consumer
compatibility remain prerequisites for enforcement. Geodata preprocessing currently emits the full result object,
including internal `_records` and `_status`; minimize this event before schema
enforcement. The Phase 0 audit found no Operations event-producing call sites; a payload
schema is not applicable unless Operations becomes a producer. These additive schemas do
not certify purpose limitation or retention, so the registry is not complete
enforcement.
The deploy-owned provisioner now fixes and validates pull delivery mode, explicit
ACK, replay policy, retry limits, pending and waiting-pull bounds, consumer replicas,
and full-payload delivery. The same source is synchronized to the platform mirror and has passed the
disposable-broker idempotency check. Contract CI now checks out the five service
repositories and runs the workspace event-source audit.

### Delegated joint review decisions (10 October 2026)

The workspace owner delegated the joint review to Volker Kerkhoff and Codex as
the only project team. Decisions are recorded in the
[joint review evidence](evidence/phase1-joint-review-2026-10-10.md).

- [x] Review and disposition all 68 source-derived fact schemas, including
      classification, intended subscriber groups, and field handling. Review
      all ten selected work-command payload shapes against current handlers.
      These are inventory/target contracts only; runtime enforcement remains
      gated on producer projections, prohibited-field/size checks, transaction
      coupling, idempotency/recovery evidence, and the Geodata preprocessing
      projection.
- [x] Accept the cluster-internal NATS trust boundary without NATS authentication
      or TLS. The broker remains ClusterIP-only; any pod with network reachability
      is trusted. Do not expose NATS outside the cluster; revisit if that boundary
      or the workload trust model changes.
- [x] Record finite byte/message/age/payload caps totaling 5 GiB of the
      existing 8 GiB PVC, with 3 GiB reserved. Volker accepts the short-sample
      limitation for the single-node scope; do not increase caps without a
      representative 30-day traffic/outage-backlog review.
- [x] Select local disposable-PVC restore/replay and database-authoritative
      redrive. Local restore/replay passed. Off-node recovery is deferred by the
      project owner; node/cluster loss can discard the bounded transport window.
      Interest retention may already have deleted acknowledged legacy records.
- [x] Select one deployment-owned create-only provisioner and prohibit relay
      topology mutation at the safe compatibility cutover. An optional fail-closed
      Helm pre-upgrade readiness hook is integrated; relay mutation changes remain
      Phase 2. Do not target the mixed legacy stream.

**Exit criteria**

- [x] Contract and subject registry covers the Phase 0 inventory and matches
      ADR-0008's fact/work classification and subject namespaces. Contracts
      tests and the five-service source audit pass; provisioned work filters
      match all ten registered work commands.
- [x] Provisioning is deterministic and create-only, drift fails closed, and
      the opt-in Helm pre-upgrade hook blocks workload upgrade on failed
      validation. Its gate is tested by chart rendering; live activation is
      intentionally deferred until Phase 2 compatibility gates pass.
- [x] Retention and local restore/replay policies have an operator runbook and
      isolated qualification. Off-node recovery is explicitly deferred for the
      current single-node scope, with the bounded transport-loss risk accepted.
- [x] Finite conservative limits and the evidence limitation are recorded;
      Volker accepts these limits for the current single-node scope. Raising
      limits requires a representative 30-day traffic and outage-backlog review.

**ChatGPT prompt — Phase 1**

```text
Work in the MyOTA multi-repository workspace to complete Phase 1 only. Read
ADR-0008, the Phase 0 event/work inventory, this plan, the joint review, the
recovery runbook, and AGENTS.md before editing. Authoritative sources are
myota-contracts for event/work contracts, myota-deploy for provisioning and
Helm, owning service repositories for source evidence, and myota-docs for
roadmaps/evidence. Treat myota-platform and organization-profile links as
synchronized mirrors; keep exact copies and links current.

Phase 1 scope is versioned envelopes, exhaustive event/work registries and
schemas, topology definition, create-only provisioning, drift/correctness checks,
recovery guidance, and isolated single-node qualification. Keep PostgreSQL as
the business/recovery authority. JetStream is a bounded transport/replay window,
not an event archive. Keep facts on the Limits-retained MYOTA_EVENTS target and
competing commands on disjoint WorkQueue-retained Activity/Geodata streams.
Keep scheduled/reconciliation work on its scheduler/database path.

The accepted trust boundary is cluster-internal ClusterIP without NATS auth or
TLS; any pod that can reach the service is trusted. Never expose it externally
without revisiting that decision. Use conservative 1/1/3 GiB stream caps,
500k/250k/500k message counts, a 30-day age cap, 1 MiB max message, file
storage, one replica, DiscardNew, and at least 3 GiB reserve on the existing
8 GiB PVC. The sampled database history covers fewer than ten days and includes
load-test traffic; Volker accepts that evidence limitation for the single-node
scope. Do not raise caps without representative 30-day serialized traffic and
outage-backlog evidence. Off-node recovery is deferred at the project owner's
direction; record the node/cluster transport-window loss risk and rely on
PostgreSQL reconciliation/redrive.

Keep the live mixed Interest-retained MYOTA_EVENTS stream and production
producer/consumer paths unchanged. The target provisioner must fail closed on
drift and must never be run against that mixed stream. Add only an opt-in,
fail-closed Helm pre-upgrade barrier; keep it disabled until the Phase 2
compatibility/cutover gate is explicitly confirmed. Runtime envelope enforcement,
payload minimization, retry/dead-letter changes, and relay mutation removal
belong to later phases. Operations remains metadata-only and cannot read message
payloads, consume, acknowledge, purge, or mutate topology.

Test contracts, source audit, provisioning idempotency/drift, bounded local PVC
restore/replay, capacity rejection, Helm lint/render, and mirrors using an
isolated namespace on the local K3s server. Do not use production resources;
remove the temporary namespace and test data afterward. If a gate cannot be
verified, record the exact gap and its later-phase owner instead of claiming it
is done. Update the plan, completion evidence, diagrams, history with the exact
prompt used, README links, To do, Work in progress, and prioritized backlog.
Mark checkboxes only when evidence supports the stated scope. Commit and push
affected work directly to each repository's main branch with explicit messages.
Report commands/results, repositories/files, mirrors, risks, remaining Phase 2
gates, and Phase 1 exit criteria.
```

### Phase 2 — Relay hardening and domain-event coverage

**Work**

- [ ] Bring the shared relay to the contract: stable event IDs, bounded payloads,
      connection/reconnect behavior, bounded concurrency, retries/backoff,
      idempotent publish, publish-ack handling, failure metrics, and dead-letter
      inspection/replay. Ensure a crash after publish acknowledgement but before
      outbox marking safely republishes the same message ID.
- [ ] Verify the core, activity, and geo relays see only their owned database and
      that retention cleanup cannot remove unpublished, dead-lettered, or
      operationally needed rows. Add indexes/partitioning only from measured need.
- [ ] Enumerate all writes in Identity, Programme, Activity, Geodata, Operations;
      route every supported event through the transactionally coupled outbox. Remove
      any process-local event dispatch for accepted cross-service work.
- [ ] Provision one durable per independently required domain-event consumer group.
      Filter to supported event subjects where feasible; list event types with no
      current subscriber in the registry instead of creating no-op consumers.
- [ ] Confirm Operations remains metadata-only and does not accidentally become a
      broker consumer with acknowledgements.

**Exit criteria**

- [ ] Inventory reconciliation finds no event write bypassing the outbox or
      unregistered event subject.
- [ ] Relay restart/retry proves no lost accepted event and safely recovers from a
      publish/mark crash: broker deduplication applies within its configured window,
      and database idempotency protects retries outside that window.
- [ ] Outbox backlog, oldest age, retries, and dead letters are observable and
      actionable for all three relays.

**ChatGPT prompt — Phase 2**

```text
Implement Phase 2 of the MyOTA NATS migration in the repositories that own the code.
Read the approved inventory and Phase 1 contract/topology first. Scope is the three
database-bound relays and complete outbox publication for every event actually
emitted by Identity, Programme, Activity, and Geodata. Include Operations only if
Phase 0 confirms event-producing call sites; do not infer emitted events from its
shared state helper.

Harden myota-deploy/services/outbox_worker.py and its source/synchronized copies as
appropriate: stable Nats-Msg-Id, publish acknowledgement before marking published,
retry/backoff, bounded connection/concurrency behavior, dead-letter
visibility/recovery, safe duplicate publish after crash, unknown-subject handling,
metrics/logging, shutdown and cluster-internal connection handling. Preserve database ownership:
core, activity, and geo relays must each use only their configured database. Do not
let the Operations status path mutate broker state.

Walk every event write and migration/recovery insert in the Phase 0 matrix. Ensure
mutations and outbox inserts are atomic and all accepted cross-service events
publish through the registered envelope/subject path. Add or update explicit
durable consumers only for required domain-event groups in this phase; each must
have clear supported-type filters, idempotency, explicit ack-after-commit,
version handling, and a poison-event path. Do not add broad no-op consumers. The
selected Limits-retained fact stream does not require a durable for every
published subject; provision durables only for independent groups with an
intended business use. Keep the four existing Geodata work durables on the
current stream until the controlled migration provisions replacements on
`MYOTA_GEODATA_WORK`.

Synchronize authoritative sources into myota-deploy/myota-platform mirrors and
update contracts/docs/operations for actual behavior. Use focused tests for relay
failure windows, unknown routes, and event coverage according to existing test
conventions. Provide an event-by-event completion matrix and evidence for relay
retry, duplicate publish safety, dead-letter visibility, and all producer owners. Do
not migrate Activity's database-polled jobs here unless the approved plan assigned
them to this phase.
```

### Phase 3 — Consumer reliability and domain-event subscribers

**Work**

- [ ] Standardize Activity notification and all other event consumers around a
      shared service-owned JetStream adapter or an explicitly documented per service
      pattern. Preserve domain ownership: Activity notification filters cover the
      approved Identity facts and the two Geodata review/status facts in the
      inventory. Do not add Programme notices or other subscriptions without an
      owner and registered consumer-group decision.
- [ ] Separate independent consumers into separate durables. Set explicit filter,
      ack wait, max deliveries, max ack pending, backoff, delivery policy, and
      concurrency based on measured handler duration and recovery needs.
- [ ] Commit domain side effect and consumer deduplication/checkpoint in one local
      database transaction when possible; acknowledge only after commit. Validate
      behavior when the database commit succeeds but ack is lost.
- [ ] Define poison-message handling that records full diagnostic envelope safely,
      avoids exposing secrets/PII in logs, notifies operations, and supports
      reviewed replay after remediation. Avoid immediately terminating errors
      without a supported recovery path.
- [ ] Keep durable consumer names stable across releases; create successor durables
      deliberately and remove obsolete durables only after old workers drain and
      backlog disposition is understood. For fact replay, use a separate durable
      with an explicit start position; never rewind a production side-effecting
      durable. Preserve the selected 30-day bounded fact replay window and use
      database state as the source for longer-term reconstruction.

**Exit criteria**

- [ ] Every intended domain-event consumer group is live, documented, independently
      deployable, idempotent, observable, and tested against redelivery/restart.
- [ ] No unsupported broad consumer determines retention accidentally.

**ChatGPT prompt — Phase 3**

```text
Implement Phase 3: standardize and complete MyOTA domain-event consumers on NATS
JetStream using the approved registry and topology. Focus on Activity notifications
and any additional subscriber groups named in the Phase 0 inventory. Do not migrate
Geodata work consumers or Activity DB jobs unless a dependency requires a small
preparatory change; keep work commands distinct from domain events.

For each durable consumer group, document its owning service, subject filter,
supported event versions, business side effects, local database, stable idempotency
key, ack boundary, retry/backoff/max-delivery policy, poison-message procedure,
shutdown/drain behavior, metrics, replica/scaling model, and durable migration
procedure. Ensure unrelated handlers never share a durable and same-group replicas
do. Where possible, commit side effect plus processed-event/checkpoint record
atomically, then ack. Handle duplicate delivery and database-commit/ack-loss
explicitly.

Replace catch-all/no-op behavior with reviewed filters and explicit event-type
dispositions. Do not make Activity own Identity/Programme/Geodata records; it may
own the selected notification projections only. Preserve Operations as read-only
status inspection. Keep stable durable names, and document how to roll to a
successor durable without dropping retained fact messages.

Update owning service code, container/deployment wiring, synchronized mirrors,
contracts, operations runbooks, and myota-docs. Add focused
delivery/restart/idempotency tests following existing conventions. Produce a
consumer-by-consumer matrix with evidence for supported versions, duplicate
delivery, poison event recovery, backlog metrics, and graceful shutdown. Mark only
verified rows complete.
```

### Phase 4 — Move Activity database-polled accepted work to JetStream

**Work**

- [ ] For each selected migratable `activity_worker.py` job kind
      (`QSO_INGESTION`, `ADIF_IMPORT`, `AWARD_RECALCULATE`, `AWARD_EVALUATION`,
      `PDF_RENDER`, `STATISTICS_REBUILD`), trace the producer, job row, side effects, retry
      semantics, payload size, idempotency key and completion state. Include
      `activity.adif.queued.v1` and related outbox writes.
- [x] Record the decision to exclude `NOTIFICATION_SEND` from JetStream migration:
      its current handler
      only changes notification state and has no external provider call. Correct or
      remove the synthetic job in the Activity implementation phase; if external
      delivery is introduced, define a separate provider-backed work command.
- [ ] Move accepted jobs to transactional outbox + dedicated command subjects and
      durable pull queues on `MYOTA_ACTIVITY_WORK` using the selected
      `myota.work.activity.*` namespace, keeping the job/resource row as domain
      status and recovery evidence. Large payloads should remain in owned storage
      and the work event should carry identifiers, not copied content.
- [ ] Use per-kind or compatible worker-group durables and bounded concurrency; do
      not put unrelated long PDF/ADIF jobs behind a single serial queue unless
      ordering is explicitly required. Preserve leases for long-running work and
      heartbeat/visibility where appropriate.
- [ ] Use one publish path during migration; do not dual-publish work to both
      systems. Define stable work IDs and database idempotency. Stop new DB-queue
      claims before translating or draining pending rows, and switch producers
      only after the durable is provisioned. Provide rollback without re-enqueueing
      completed jobs.
- [ ] Keep necessary periodic maintenance/reconciliation tasks in schedulers when
      they are timer-triggered rather than event-driven; record why they are not
      JetStream messages.

**Exit criteria**

- [ ] Each of the six selected Activity jobs is JetStream-backed; the excluded
      `NOTIFICATION_SEND` state-only job is corrected or removed. No job is
      acknowledged/completed before durable side effects.
- [ ] Cutover and rollback preserve business invariants and prevent duplicate
      effects under at-least-once delivery.
- [ ] Activity job latency, queue age, failure, retry, and dead-letter states are
      visible in service and operations dashboards.

**ChatGPT prompt — Phase 4**

```text
Implement Phase 4: migrate the six selected Activity asynchronous jobs from
PostgreSQL polling to the `MYOTA_ACTIVITY_WORK` JetStream WorkQueue stream,
following ADR-0008 and the Phase 1 contracts. Do not migrate the synthetic
`NOTIFICATION_SEND` job: correct/remove its state-only transition in Activity; if
external delivery is introduced, model it as a separate provider-backed command.
Inspect activity_repository.py, activity_worker.py,
outbox writes, job migrations/schema, Activity API job producers, deployment
manifests, and all recovery/retention paths before editing.

Inventory and handle these selected job kinds individually: QSO_INGESTION,
ADIF_IMPORT, AWARD_RECALCULATE, AWARD_EVALUATION, PDF_RENDER, and
STATISTICS_REBUILD. Map each to a versioned `myota.work.activity.*` subject and
one durable consumer group shared by its worker replicas. Do not include
`NOTIFICATION_SEND` in the work stream because it has no external delivery side
effect today. Preserve resource/job status in `myota_activity`. Persist work
request/outbox atomically with the accepted state transition; publish identifiers
and bounded metadata rather than large content. Keep blob data in the configured
object store. Ensure long-running tasks use suitable ack wait/progress/lease
strategy and independent concurrency so unrelated heavy work does not block other
kinds.

Implement at-least-once-safe processing: stable job/event ID, domain idempotency,
explicit ack after committed completion/failure state, bounded redelivery/backoff,
visible poison/dead-letter handling, graceful drain, and startup recovery. Design a
controlled cutover from DB polling, with one publish path, stable work IDs,
old-job drain or translation, database idempotency, metrics, and rollback. Do not
dual-publish, delete job history, or claim exactly-once broker delivery. Keep
scheduled retention/reconciliation tasks on their scheduler/recovery paths as
selected in ADR-0008.

Update Activity source, contract/subject registry, deployment and synchronized
myota-deploy/myota-platform copies, tests, Activity README and central MyOTA docs.
Validate each job kind with focused unit/integration evidence and report
cutover/rollback instructions, remaining DB polling, job age/backlog visibility, and
any blocked handler. Do not move Geodata work into Activity ownership.
```

### Phase 5 — Reconcile Geodata queues and cross-service recovery

**Work**

- [ ] Move the four Geodata work kinds from their current `myota.geodata.*`
      subjects in `MYOTA_EVENTS` to disjoint `myota.work.geodata.*` subjects in
      `MYOTA_GEODATA_WORK`. Provision the new WorkQueue durables before switching
      routing; move pending rows through one publish path, without dual-publishing.
      Drain legacy messages and consumers only after rollback and database recovery
      checks pass.
- [ ] Validate each replacement Geodata durable against the registered work
      contract, shared provisioning, deployment replicas and Operations view.
- [ ] Confirm consumer side effects and processed-event/checkpoint state are atomic
      where possible; inspect `_consume` behavior for transient errors, max
      delivery, ack/nak/term and DLQ compatibility. Ensure long import jobs use
      heartbeat, leases, bounded inflight, and result recovery.
- [ ] Keep stale cancellation and pending deletion reconcilers as repair loops that
      can restore work if a message is missing/expired; ensure they do not create
      duplicate side effects. Verify location enrichment's request ID/geometry hash
      protects against stale provider responses.
- [ ] Verify current activity cascade-deletion cross-service sequencing,
      compensation and partial-failure recovery remain correct after Activity work
      migration.
- [ ] Prove all accepted Geodata work is recoverable from either outbox/JetStream or
      durable domain job state within documented age bounds. Run outage and restore
      scenarios with Operations status/read-only observability.

**Exit criteria**

- [ ] Each Geodata queue has documented scaling, retry, DLQ, recovery and retention
      behavior; all four replacement durables on `MYOTA_GEODATA_WORK` are validated
      in each environment and the legacy route is retired safely.
- [ ] Broker loss, worker restart, delayed ack and duplicate delivery do not lose or
      repeat domain effects.
- [ ] Recovery loops are bounded, observable, and do not become a second primary
      dispatch path.

**ChatGPT prompt — Phase 5**

```text
Implement Phase 5: move the four Geodata work kinds onto the selected
`MYOTA_GEODATA_WORK` WorkQueue stream and qualify all cross-service recovery paths.
The current durable pull consumers are preprocessing, import promotion, entity
deletion, and location enrichment. Their current `myota.geodata.*` subjects are
captured by `MYOTA_EVENTS`; use disjoint `myota.work.geodata.*` subjects for the
new stream. Read the event contract, Geodata architecture/runbooks, Phase 0
inventory, and ADR-0008 first.

Verify the producer transaction, new work subject, provisioned durable/filter/config,
worker replica model, explicit ack policy, ack wait/max-deliver/max-ack-pending,
database idempotency/checkpoint/lease, long-work heartbeat, failure/dead-letter
behavior, and recovery source for each queue. Inspect transient failures and ensure
retryable lease contention is deferred rather than irreversibly terminated. Confirm
acknowledgement follows durable side effects. Preserve location request ID and
geometry-hash recheck, deletion authorization and Activity impact sequencing, import
cancellation semantics, and database reconciliation loops as repair mechanisms
rather than competing primary queues.

Provision replacement durables before changing routes. Do not dual-publish. Drain
or translate pending legacy work through one path, then remove the old filters only
after rollback and database recovery checks pass. Keep Operations read-only.

Exercise failure scenarios: publish acknowledgement lost before outbox mark; worker
crash before/after database commit; database outage; delayed import beyond ack wait;
duplicate message; stream age expiry; durable recreation; broker restore;
cancellation racing with processing; and Activity/geodata partial deletion failure.
Fix issues within the repositories that own them, update synchronized deployment
mirrors, add focused tests/evidence, and refresh operations metrics/alerts and
recovery instructions. Operations remains read-only. Report each queue and failure
scenario as pass/fail/open with evidence and avoid claiming production qualification
without production-like results.
```

### Phase 6 — Shadow, canary, cutover, and retirement

**Work**

- [ ] Establish baseline metrics: accepted-to-published latency, outbox oldest
      age/depth, publish retries, broker storage, consumer pending/ack-pending,
      redelivery, oldest event age, handler duration/failure, DLQ volume, and
      business completion latency.
- [ ] Run shadow validation by comparing event/job IDs and resulting state without
      executing duplicate side effects. Prefer replay/verification tooling or
      non-mutating observers. Never dual-consume a mutating command without shared
      idempotency protection.
- [ ] Canary one producer/consumer group or environment, then expand by service.
      Gate promotion on no unexplained event-count divergence, bounded lag, no
      unexpected DLQ growth, and successful restart/recovery drills.
- [ ] Cut over producers atomically to the registered route; disable old dispatch
      only after JetStream path proves healthy. Drain existing DB queue rows and
      outbox backlog before removing old worker claims. Keep rollback switches
      time-bounded and observable.
- [ ] Remove retired code/config/durables only after confirming all versions are
      drained, documenting archived evidence, and checking no stale consumer can
      compete with the new durable.

**Exit criteria**

- [ ] All producers and required consumers use JetStream according to the registry;
      remaining exceptions have named owner, rationale, and review date.
- [ ] Rollback and recovery have been rehearsed; no event or job was lost during
      cutover and no business effect duplicated.
- [ ] Production operations accept dashboards, alerts, runbooks, backup/restore, and
      backlog/DLQ ownership.

**ChatGPT prompt — Phase 6**

```text
Prepare and execute the approved Phase 6 shadow/canary/cutover/retirement plan for
the MyOTA NATS migration. Before changing deployment or runtime switches, read every
prior phase's completion evidence and the production rollout approval documented by
the user/team. If any exit gate is unverified, do not cut over that service;
complete independent safe preparation and report the exact blocker.

Build a per-service rollout matrix for Identity, Programme, Activity, Geodata,
Operations, and shared deployment: old path, new subject/durable, baseline counts,
canary scope, success thresholds, rollback trigger, drain condition, responsible
owner, and observed evidence. Shadow comparison must not execute duplicate mutating
work. Never dual-consume a command unless both paths share proven idempotency and a
bounded migration window. Canary progressively, measure event/job ID reconciliation,
accepted-to-published delay, consumer lag/oldest age, retries, DLQ, worker errors,
business completion and broker capacity. Verify Operations continues read-only
access.

Only after gates pass, perform the authorized cutover: switch producers, stop old
claims after acceptance is confirmed, drain legacy queues, reconcile
outbox/DLQ/backlog, then retire obsolete code and durables. Keep a tested rollback
route until the agreed observation window ends. Update runbooks, service READMEs,
contracts, deployment docs, central plan status, and mirrors with actual evidence
and dates. Do not state complete based on deployment success alone. End with a
signed-off event/consumer coverage matrix, any exceptions with owner/review date,
and recovery/rollback evidence.
```

## Cross-phase verification and rollout gates

These are required evidence items, not a promise that current code already
passes them. Use the repositories' existing quality and integration workflows;
do not treat unit-only coverage as broker/recovery proof.

- **Contract and routing:** all event/work types have schema, owner, subject,
  intended durable groups, payload limits, version handling and compatibility
  path. Unknown subject/version behavior is intentional and observable.
- **Atomic publication:** force transaction rollback after attempted state
  mutation and verify no outbox event; accept and crash each relay boundary to
  verify retry and stable broker deduplication.
- **Consumer reliability:** duplicate delivery, consumer restart, ack loss,
  database failure, poison message, redelivery exhaustion, DLQ replay and
  durable migration each preserve business invariants.
- **Backpressure and isolation:** saturate one work group, slow one handler,
  grow broker/outbox backlog, and prove bounded memory/connections and isolation
  of unrelated event/work groups.
- **Retention and recovery:** validate no required message is removed before
  consumer completion; restore broker and consumer state; demonstrate bounded fact
  replay through an isolated durable and work recovery/redrive from database
  records and reconcilers.
- **Cluster boundary:** keep NATS behind its cluster-internal ClusterIP service.
  Under the accepted decision, NATS authentication and TLS are not required while
  all pods with network reachability are trusted. Revisit before external exposure
  or a change to the workload trust model. Prevent unnecessary sensitive payload
  and log exposure; keep Operations read-only in its application behavior.
- **Observability:** verify outbox depth/age and publish/dead-letter metrics for
  core/activity/geo; per-stream/durable pending, ack-pending, redelivery and
  oldest age; handler success/failure/latency; and business job completion.
  Alert thresholds must have runbooks and avoid high-cardinality labels.
- **Deployment:** compose and Helm use equivalent subject/consumer config;
  workers have readiness/liveness behavior appropriate to broker availability,
  graceful drain, database pool limits, disruption/restart behavior, and
  independent scaling. Test provisioning and rollout ordering so streams and
  required durables are ready before associated workers rely on them. Validate
  bounded fact replay under Limits and database recovery for work subject to
  WorkQueue expiry.
- **Inventory reconciliation:** compare registry against static event writes,
  runtime streams/consumers, deployed worker processes, and sampled event/job
  IDs. A zero backlog alone does not prove event coverage.

## Rollback principles

- Roll back consumers before producers only when their durable backlog remains
  intact and an old implementation can safely resume the same durable or a
  reviewed successor durable.
- Never switch both the old DB poller and JetStream consumer to execute the same
  non-idempotent job without shared idempotency enforcement.
- Do not lower retention, delete durables, purge streams, or clear outbox/DLQ
  rows as a routine rollback step.
- Record the last reconciled event/job ID and database state before resuming a
  legacy path. Reconcile completion state before replay.
- Operations status remains inspection-only; broker mutations use a controlled
  deployment/recovery process with audit evidence.

## Repository ownership and documentation updates

- `myota-contracts`: event/work schemas, version and compatibility contracts.
- Each service repository: its outbox writes, worker handlers, local consumer
  idempotency and domain recovery.
- `myota-deploy`: broker service exposure, relay/worker deployment,
  rollout and operational controls.
- `myota-platform`: synchronized integration bootstrap mirrors and end-to-end
  wiring, not an independent domain source.
- `myota-docs`: authoritative cross-service plan, ADR, event matrix, runbooks,
  evidence and status reconciliation.

At completion, update this plan with phase status, exact verification evidence,
rollout date/environment, and unresolved exceptions. Update the event contract,
architecture, repository map, operations guide, relevant service READMEs, and
the public roadmap when their claims/topology/status change. Keep synchronized
mirrors in sync with their owners and do not mark a phase complete on the basis
of code changes alone.
