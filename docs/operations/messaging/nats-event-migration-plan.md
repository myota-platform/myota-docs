# NATS JetStream event and work-queue migration plan

**Status:** Phase 0 inventory and decision record complete. Phases 1–6 cover
runtime implementation and qualification. Phase 1 contract/provisioning
preparation is in progress; no phase after Phase 0 is claimed complete.

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
The event contract remains a representative list, not a machine-checked event
catalogue. The workspace owner selected the documented topology; implementation,
operational qualification, and unresolved evidence gates remain open.

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
   `MaxAge`, `MaxBytes`, `MaxMsgs`, and `MaxMsgSize` with `DiscardNew`; choose
   numeric work limits from measured load and recovery objectives. Keep one
   replica on today's single-server deployment; use three only after a
   three-server JetStream cluster is deployed. Qualify backup/restore and
   replay for each environment.
6. JetStream is delivery infrastructure, not the permanent event archive.
   Choose retention according to the required processing and recovery window;
   keep domain history and outbox/dead-letter evidence in service-owned storage.
   Retained facts can be replayed through a new isolated durable with an
   explicit start position; work is redriven from its owning database with the
   same work ID. Acknowledged-event replay must use an explicit supported path.
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
10. Keep the consumer APIs/web clients out of direct NATS access. Services and
    workers alone hold the narrow credentials they require.

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

**Work**

- [x] Produce the exhaustive producer/event/subject/consumer matrix across all five
      services, outbox relay, existing DB job queue, recovery migrations, and
      deployment mirrors.
- [x] Identify every accepted asynchronous request path, including Activity's
      PostgreSQL `claim_job` worker loop. Label each as domain event, work command,
      scheduled/reconciliation task, or synchronous request.
- [x] For each event, document the owning service, event schema/version, transaction
      boundary, current and target subject, independent durable groups, idempotency
      key, retry/dead-letter policy, retention/replay requirement, and API/business
      impact if delayed or lost.
- [x] Resolve whether the shared `MYOTA_EVENTS` stream remains the target or whether
      domain events and work queues need separate streams. Compare Interest versus
      WorkQueue retention for each class, cross-service blast radius, independent
      replay/retention needs, deployment complexity, and migration safety. Do not
      split streams merely for naming symmetry.
- [x] Add an ADR or amend the event contract with the selected topology, subject
      naming, ownership, compatibility and deprecation policy. Carry uncovered
      consumers and unclassified event types as gates on the affected producer or
      consumer rollout; Phase 1 contract work resolves their dispositions.
- [x] Create the evidence register below with proposed authoritative repository
      owners, closure phases, and the producer/consumer cutover each gap gates.
- [x] Confirm the assignments with the workspace owner and record the individual
      assignee in each owning repository's work item. Phase 1 contract/topology
      work may proceed while evidence is being closed; a producer or consumer path
      must not change until its listed gate is met.

**Exit criteria**

- [x] Every outbox event write and database-backed asynchronous job is accounted
      for; each has a disposition and owner.
- [x] Stream/retention topology and activity-job migration scope are selected in
      documentation before implementation changes begin.
- [x] Mirrored files and authoritative repositories are explicitly identified.
- [x] Every remaining evidence gap is mapped to a proposed authoritative repository,
      closure phase, and explicit cutover gate.
- [x] The workspace owner confirms the assignments; individual assignees are
      recorded in the owning work items. Record any future risk acceptance there
      with its expiry/review point and recovery plan.

#### Phase 0 evidence ownership and gates

The evidence gaps are not a blanket blocker to starting Phase 1 contract and
topology work: defining the contract, registry, limits, and safe provisioning is
part of Phase 1. They are gates on the runtime change that depends on them. Phase 1
may implement and validate changes in isolation, but must not activate live stream
configuration or change producer/consumer behavior before the relevant gate.

The workspace owner confirmed the assignments in this task. This workspace has no
separate service teams: Volker Kerkhoff (`@kerk1v`) is the named individual and
GitHub assignee for the evidence work items; Codex is the pairing agent and is not
a separate GitHub account. Accountable source repositories below follow the
[repository map](../../architecture/repository-map.md); mirrors are not owners.
Each work item records the evidence, gate, and assignee.

| Inventory gap | Accountable repository owner | Close before | Individual assignee and work item |
|---|---|---|
| 1. Envelope schema and compatibility rules | `myota-contracts` (with event-producing service owners) | Phase 2 publishes the new envelope or subject contract | Volker Kerkhoff (`@kerk1v`); [contracts #1](https://github.com/myota-platform/myota-contracts/issues/1) |
| 2. Intended subscriber groups and event dispositions | `myota-contracts`, with each relevant service owner (`myota-identity-service`, `myota-programme-service`, `myota-activity-service`, `myota-geodata-service`) | Phase 3 enables or changes a consumer group | Volker Kerkhoff (`@kerk1v`); [contracts #1](https://github.com/myota-platform/myota-contracts/issues/1), [Identity #1](https://github.com/myota-platform/myota-identity-service/issues/1), [Programme #1](https://github.com/myota-platform/myota-programme-service/issues/1), [Activity #1](https://github.com/myota-platform/myota-activity-service/issues/1) |
| 3. Activity notification/checkpoint atomicity and synthetic job state | `myota-activity-service` | Phase 3 changes the notification consumer; Phase 4 removes/corrects the synthetic job | Volker Kerkhoff (`@kerk1v`); [Activity #1](https://github.com/myota-platform/myota-activity-service/issues/1) |
| 4. Geodata transaction coupling and source recovery for fallback/upload paths | `myota-geodata-service` | Phase 2 changes affected fact publication or Phase 5 changes affected work routing | Volker Kerkhoff (`@kerk1v`); [Geodata #2](https://github.com/myota-platform/myota-geodata-service/issues/2) |
| 5. Activity worker idempotency and missing `Idempotency-Key` behavior | `myota-activity-service` | Phase 4 switches the affected job producer/worker | Volker Kerkhoff (`@kerk1v`); [Activity #1](https://github.com/myota-platform/myota-activity-service/issues/1) |
| 6. Relay/consumer dead-letter redrive and database retention | `myota-deploy` coordinates relay/runbook and physical-database retention evidence with the service owners | Before enabling a cutover that relies on redrive or cleanup of those records | Volker Kerkhoff (`@kerk1v`); [deploy #3](https://github.com/myota-platform/myota-deploy/issues/3) |
| 7. Geodata migration-015 recovery identity and concurrent-startup behavior | `myota-geodata-service`, with `myota-deploy` for migration rollout | Phase 5 switches Geodata work to `MYOTA_GEODATA_WORK` | Volker Kerkhoff (`@kerk1v`); [Geodata #2](https://github.com/myota-platform/myota-geodata-service/issues/2) |
| 8. Measured limits, backup/restore, permissions, and deployed NATS config | `myota-deploy`; `myota-operations-service` supplies inspection/observability evidence | Before stream provisioning is qualified; production evidence before Phase 6 rollout | Volker Kerkhoff (`@kerk1v`); [deploy #3](https://github.com/myota-platform/myota-deploy/issues/3), [Operations #1](https://github.com/myota-platform/myota-operations-service/issues/1) |
| 9. Producer-to-consumer coverage and registry checks | `myota-contracts` coordinates; owning service and `myota-deploy` test owners supply evidence | Before Phase 2/3 coverage is declared complete and before Phase 6 retirement | Volker Kerkhoff (`@kerk1v`); [contracts #1](https://github.com/myota-platform/myota-contracts/issues/1) |

For each gap, the linked work item includes the individual assignee and target
gate. Record evidence links, residual risk, and mitigation as the work progresses.
If evidence cannot be completed
before its gate, the affected service owner and `myota-deploy` must record explicit
risk acceptance, expiry/review point, and recovery plan. A blanket “accepted” entry
without those fields does not satisfy the gate. Documentation link cleanup is a
separate maintenance item and is not a runtime migration gate.

**ChatGPT prompt — Phase 0**

```text
Work in the MyOTA multi-repository workspace. Create an exhaustive, evidence-backed
inventory for moving all outbox events and asynchronous consumers to NATS JetStream
streams and durable queues. Do not change runtime code in this phase.

Read ADR-0008 before reviewing the topology. The selected target is one bounded
Limits fact stream and separate Activity/Geodata WorkQueue streams. Verify that
repository evidence supports the recorded decision; do not reopen it based only
on preference. Report concrete contradictions or evidence that requires changing
the decision.

Inspect the authoritative repositories: myota-identity-service,
myota-programme-service, myota-activity-service, myota-geodata-service,
myota-operations-service, myota-contracts, myota-deploy, myota-platform, and
myota-docs. Treat deploy/platform copies as synchronized mirrors and identify their
source owners. Inspect migrations, recovery SQL, event creation helpers/call sites,
outbox relay/routing, consumer subscriptions/handlers, workers, compose/Helm
deployment, docs, and tests.

Return a table with one row per event type and asynchronous work type: producer
service/database, transaction/write location, current event type and subject,
envelope/schema version, intended consumer groups, current durable/filter,
processing and idempotency boundary, retry/dead-letter behavior, current source of
truth, retention/replay need, deployment owner, and missing evidence. Distinguish
committed domain facts, competing-consumer work commands, scheduled/reconciliation
work, and synchronous operations. Include Activity database-polled jobs and identify
whether each should migrate to JetStream or remain as a justified exception.

Specifically assess the current shared file-backed MYOTA_EVENTS stream, Interest
retention, currently provisioned Activity notification durable and four Geodata
work durables, the three database-specific outbox relays, operations read-only
inspection boundary, and existing replay limitations. Reconcile current behavior
with ADR-0008's selected topology and record its tradeoffs; do not assume every
event is a command or that JetStream is an event archive.

Edit only myota-docs in this phase: reconcile the inventory, migration plan, and
ADR-0008; resolve remaining evidence gates or identify their owners. Keep claims
labeled current/selected and cite exact repository paths. Do not mark runtime
implementation complete. Report missing evidence and Phase 0 exit criteria.
```

### Phase 1 — Contracts, topology, provisioning, and operational safety

**Current status (10 October 2026):** the first registry/schema and create-only
provisioner artifacts are implemented and their focused local checks pass. No
live topology or producer/consumer path changed. Payload schemas, least-privilege
credentials, measured capacity limits, restore/replay qualification, and removal
of legacy relay provisioning remain open. An isolated host-cluster broker test
created all target streams/durables, passed an idempotent second run, and rejected
configuration drift; its temporary namespace was removed. The 10 October follow-up
also validates the expanded durable consumer configuration on a disposable broker.
See the [Phase 1 evidence record](evidence/phase1-contract-topology-2026-10-09.md)
and [current/target topology diagrams](../../architecture/diagrams/nats-event-migration.md).

**Verified preparation (does not satisfy the phase exit criteria):**

All four implementation PRs are merged: [contracts #2](https://github.com/myota-platform/myota-contracts/pull/2),
[deploy #4](https://github.com/myota-platform/myota-deploy/pull/4),
[platform mirror #1](https://github.com/myota-platform/myota-platform/pull/1),
and [organization profile #1](https://github.com/myota-platform/.github/pull/1).
Merge records and CI results are in the [Phase 1 evidence record](evidence/phase1-contract-topology-2026-10-09.md).
Merging does not authorize live provisioning or runtime changes. The platform
unit-test and Ruff jobs passed; the image-build job could not obtain a Docker Hub
token on two attempts and remains unverified.

- [x] Add a contracts-owned registry for all 68 inventory facts, six legacy
      Geodata work/recovery event types mapped to four commands, and six
      proposed Activity work commands, with per-fact outer-envelope schemas.
- [x] Add create-only provisioning for the three target streams and ten work
      durables; verify initial creation, idempotent repeat, and drift rejection
      on an isolated broker. Capacity values used in that test are not
      production limits.

**Work**

- [ ] Define/enforce the selected immutable envelope with `envelopeVersion: 1`,
      required fields, JSON encoding, timestamp/UUID semantics, trusted correlation
      and optional causation propagation, payload size limits, compatibility rules,
      and unknown-version behavior. Keep envelope version separate from event type
      suffix `.vN`; keep relay attempts out of the envelope.
- [ ] Define the selected subject convention and registry: facts use
      `myota.events.<eventType>` with dotted tokens preserved; work uses disjoint
      `myota.work.activity.*` and `myota.work.geodata.*` namespaces. Add checked-in
      per-event JSON Schemas and subscriber dispositions in `myota-contracts`.
      Explicit work subjects must map to provisioned, non-overlapping durable
      filters. Reject unknown routing before marking an event published; make the
      failure operator-visible and actionable.
- [ ] Implement safe stream/consumer provisioning as a controlled deployment step or
      idempotent reconciler with drift detection. Avoid multiple relay replicas
      racing to mutate stream configuration. Validate all correctness-sensitive
      consumer settings, including replay, waiting-pull, delivery, and storage
      behavior. Apply least-privilege NATS credentials per relay and worker role.
- [ ] Implement the selected stream topology: bounded Limits retention on
      `MYOTA_EVENTS`, WorkQueue retention on the Activity and Geodata work streams,
      finite limits and `DiscardNew`. Derive numeric caps from measured traffic and
      recovery objectives. Keep replicas at one on a single server; qualify three
      replicas only with a three-server cluster. Define backup, restore, replay, and
      redrive procedures; validate outage, disk pressure, consumer deletion, stream
      restore, and consumer recreation behavior in local and production-like
      environments.
- [ ] Add a consumer registry and contract fixtures so event publishers cannot add a
      subject without updating schema, intended subscriber, durable provisioning,
      docs, and compatibility checks.

The machine-readable registry currently covers 68 domain facts and ten selected
work commands. The six current Geodata work/recovery event types map to four
target commands. Envelope wrappers exist, but event payload fields remain pending
repository evidence review; do not treat this draft registry as complete enforcement.
The deploy-owned provisioner now fixes and validates pull delivery mode, explicit
ACK, replay policy, retry limits, pending and waiting-pull bounds, consumer replicas,
and full-payload delivery. The same source is synchronized to the platform mirror and has passed the
disposable-broker idempotency check.

**Exit criteria**

- [ ] Contract and subject registry covers the Phase 0 inventory and matches
      ADR-0008's fact/work classification and subject namespaces.
- [ ] Provisioning is deterministic, least-privilege, observable, and safe before
      first publish; incompatible drift fails deployment/readiness clearly.
- [ ] Retention and restore/replay policies have an operator runbook and evidence.

**ChatGPT prompt — Phase 1**

```text
Implement the selected Phase 1 NATS contract and topology decisions recorded in
myota-docs/docs/operations/messaging/nats-event-migration-plan.md and ADR-0008. First read
the Phase 0 inventory and repository ownership map. Do not expand scope beyond the
approved topology.

Review the Phase 0 evidence ownership and gates table. Contract, schema, registry,
and isolated provisioning work may proceed, but do not apply live stream changes
or change producer/consumer behavior until the relevant gap is closed or the
specified owners record the required time-bounded risk acceptance and recovery
plan.

Update the authoritative myota-contracts event documentation/schema and any required
owning-service helpers. Define the versioned envelope, stable message ID, subject
naming/registry, event versus work-command classification, consumer-group naming,
unknown-version behavior, payload bounds, correlation/causation fields, and backward
compatibility rules. Update synchronized myota-platform/myota-deploy contract copies
only through their documented sync process.

Make JetStream stream and durable-consumer provisioning deterministic and
drift-checked. Register every producer subject, but provision durables only for
independent consumer groups with an intended business use; do not create broad
no-op consumers for Limits-retained facts. Provision required consumers before
relying on their processing. Implement the single controlled provisioner specified
in ADR-0008; relay replicas must not concurrently mutate stream configuration.
Validate all correctness-sensitive consumer settings.
Add least-privilege credentials and deployment configuration for each relay/worker
role. Add registry/contract checks that fail when a producer subject lacks schema,
owner, consumer disposition, provisioning and documentation.

Document retention limits, restore/replay and recovery procedures; preserve
PostgreSQL as system of record and Operations as read-only broker inspection. Update
myota-docs and operator docs. Add focused tests for routing, schema compatibility,
provisioning drift, and unknown subjects only where the repository's existing test
conventions support them. Keep mirrors synchronized and list every changed
repo/file.

Before editing, state the approved topology you found. If the ADR and plan conflict
or the Phase 0 decision is absent, stop runtime changes and report the conflict with
file references. At completion report evidence and remaining gates; do not claim
later phases complete.
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
metrics/logging, shutdown and credential handling. Preserve database ownership:
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
- **Security:** validate service-specific NATS credentials, TLS/auth policy,
  subject permissions, secret rotation, and no browser or web-client NATS
  access. Prevent event payload/log exposure of credentials or unnecessary PII.
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
- `myota-deploy`: broker configuration, credentials, relay/worker deployment,
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
