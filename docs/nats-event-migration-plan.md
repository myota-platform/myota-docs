# NATS JetStream event and work-queue migration plan

**Status:** proposed implementation plan; no migration phases in this document are
claimed complete.

**Progress tracking:** leave items unchecked until evidence is available; mark
`[x]` only when the work is verified. A phase is complete only after all its
exit-criteria items are checked and the evidence is recorded.

**Scope:** all transactional outbox events and asynchronous consumers across
MyOTA services, including the remaining database-polled activity jobs where
they are the consumer side of accepted asynchronous work.

**Owner:** cross-repository change coordinated through `myota-docs`; code changes
belong in each owning service and deployment/contracts repository.

## Goal

Make NATS JetStream the durable, observable delivery layer for every
cross-service event and accepted asynchronous work item. Keep the transactional
outbox as the atomic database-to-broker handoff: domain mutation and outbox row
must commit together, and a relay publishes with a stable message ID. Consumers
must be independently deployable, horizontally scalable where appropriate,
idempotent, and safe under at-least-once delivery.

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
| Activity / `myota_activity` | Activation/QSO lifecycle, ADIF queue, activity cascade deletion, award definition/request/issuance/rendering, and other activity/award events. | `activity-outbox` relay. | Activity notification consumer subscribes to broad domain-event interest. Separately, `activity_worker.py` still polls database jobs for ADIF, award recalculation/evaluation, PDF rendering, statistics rebuild, and notification delivery. Decide and migrate these accepted asynchronous jobs to explicit JetStream work subjects, or record a justified exception with an owner and retirement condition. |
| Geodata / `myota_geo` | Import lifecycle, validation/promotion, entity review/change/deletion, location enrichment, cancellation, recovery, and operational events. | `geo-outbox` relay. Generic events use `myota.events.<event_type with dots replaced by underscores>`; work dispatch uses allowlisted `payload.natsSubject`. | `geodata_import_worker.py` has durable pull consumers: `geodata-preprocessing-v1`, `geodata-import-processing-v2`, `geodata-entity-deletion-v1`, and `geodata-location-enrichment-v1`, plus stale-cancellation and pending-deletion database reconcilers. Keep reconciliation as recovery, not a second normal work queue. |
| Operations / `myota_core` | The shared state adapter has an outbox/event helper, but a source scan found no Operations event-producing call sites. Confirm this during Phase 0 rather than treating helper support as emitted events. | Shared `core-outbox` relay can read the core database outbox; no Operations-specific event stream was identified. | Operations reads JetStream stream/consumer status and stores samples; it is deliberately read-only and is not a business-event consumer. Preserve this boundary. |

The relay is currently shared code in `myota-deploy/services/outbox_worker.py`
and is deployed once per physical database (`core`, `activity`, `geo`). It
provisions one shared file-backed `MYOTA_EVENTS` stream with Interest retention
and known durable filters before publishing. The durable set currently includes
Activity notifications plus the four Geodata work filters. This means the
effective broker retention and rollout rules are correctness-critical: a
subject without a matching durable interest can be discarded, and acknowledged
messages are not an event archive. PostgreSQL domain state, outbox/dead-letter
records, and consumer idempotency/checkpoint state remain authoritative.

### Event/work inventory to close in Phase 0

The event contract lists representative versioned event names, but it is not yet
a machine-checked exhaustive catalogue. Phase 0 must extract all event writes
from service code, SQL migrations/recovery migrations, and test fixtures; map
each to producer database, subject, schema/version, intended consumer(s),
retention needs, and owner; and distinguish notification-only facts from
commands/work requests. Include static review of generated/synchronized copies
in `myota-platform` and `myota-deploy`, which are mirrors rather than owners.

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
   process-local list, executor, or best-effort publish.
2. The relay publishes the complete envelope (event ID/type/time, producer,
   aggregate identity, correlation/causation context, payload, schema version)
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
5. Stream/consumer provisioning is controlled and validated before publishing
   to new subjects. In Interest retention, establish all required consumers
   before a producer emits a new subject. Define and test how consumer filters,
   stream subjects, retention, max age/bytes/messages, replicas, storage,
   account limits, backup/restore, and replay are managed in each environment.
6. JetStream is delivery infrastructure, not the permanent event archive.
   Choose retention according to the required processing and recovery window;
   keep domain history and outbox/dead-letter evidence in service-owned storage.
   Acknowledged-event replay must use an explicit supported source/replay path.
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

## Phased implementation

### Phase 0 — Complete inventory and decision record

**Work**

- [ ] Produce the exhaustive producer/event/subject/consumer matrix across all five
      services, outbox relay, existing DB job queue, recovery migrations, and
      deployment mirrors.
- [ ] Identify every accepted asynchronous request path, including Activity's
      PostgreSQL `claim_job` worker loop. Label each as domain event, work command,
      scheduled/reconciliation task, or synchronous request.
- [ ] For each event, document the owning service, event schema/version, transaction
      boundary, current and target subject, independent durable groups, idempotency
      key, retry/dead-letter policy, retention/replay requirement, and API/business
      impact if delayed or lost.
- [ ] Resolve whether the shared `MYOTA_EVENTS` stream remains the target or whether
      domain events and work queues need separate streams. Compare Interest versus
      WorkQueue retention for each class, cross-service blast radius, independent
      replay/retention needs, deployment complexity, and migration safety. Do not
      split streams merely for naming symmetry.
- [ ] Add an ADR or amend the event contract with the approved topology, subject
      naming, ownership, compatibility and deprecation policy. Flag uncovered
      consumers and unclassified event types as blockers to Phase 1.

**Exit criteria**

- [ ] Every outbox event write and database-backed asynchronous job is accounted
      for; each has a disposition and owner.
- [ ] Stream/retention topology and activity-job migration scope are approved in
      documentation before implementation changes begin.
- [ ] Mirrored files and authoritative repositories are explicitly identified.

**ChatGPT prompt — Phase 0**

```text
Work in the MyOTA multi-repository workspace. Create an exhaustive, evidence-backed
inventory for moving all outbox events and asynchronous consumers to NATS JetStream
streams and durable queues. Do not change runtime code in this phase.

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

Specifically assess the shared file-backed MYOTA_EVENTS stream, Interest retention,
currently provisioned Activity notification durable and four Geodata work durables,
the three database-specific outbox relays, operations read-only inspection boundary,
and existing replay limitations. Recommend a target stream/retention/subject
topology with tradeoffs; do not assume that every event is a command or that
JetStream is an event archive.

Edit only myota-docs in this phase: add the inventory and decision proposal to the
NATS migration plan, and update docs/README.md and repository README links if
needed. Keep claims labeled proposed/current and cite exact repository paths. Do not
mark implementation complete. Report missing evidence and Phase 0 exit criteria.
```

### Phase 1 — Contracts, topology, provisioning, and operational safety

**Work**

- [ ] Define/enforce event envelope schema, required fields, JSON encoding,
      timestamp/UUID semantics, correlation and causation propagation, payload size
      limits, compatibility rules, and unknown-version behavior.
- [ ] Define canonical subject mapping. Replace implicit global dot-to-underscore
      assumptions with a reviewed, versioned registry or a documented deterministic
      convention. Explicit work subjects must be allowlisted and map to provisioned
      durable consumers. Reject unknown routing before marking an event published;
      make the failure operator-visible and actionable.
- [ ] Implement safe stream/consumer provisioning as a controlled deployment step or
      idempotent reconciler with drift detection. Avoid multiple relay replicas
      racing to mutate stream configuration. Validate complete consumer config, not
      only filter and ack policy. Apply least-privilege NATS credentials per relay
      and worker role.
- [ ] Set retention/resource limits and backups/replay procedure based on Phase 0
      decisions. Validate outage, disk pressure, consumer deletion, stream restore,
      and consumer recreation behavior in local and production-like environments.
- [ ] Add a consumer registry and contract fixtures so event publishers cannot add a
      subject without updating schema, intended subscriber, durable provisioning,
      docs, and compatibility checks.

**Exit criteria**

- [ ] Contract and subject registry covers the Phase 0 inventory.
- [ ] Provisioning is deterministic, least-privilege, observable, and safe before
      first publish; incompatible drift fails deployment/readiness clearly.
- [ ] Retention and restore/replay policies have an operator runbook and evidence.

**ChatGPT prompt — Phase 1**

```text
Implement the approved Phase 1 NATS contract and topology decisions recorded in
myota-docs/docs/nats-event-migration-plan.md and the approved event ADR. First read
the Phase 0 inventory and repository ownership map. Do not expand scope beyond the
approved topology.

Update the authoritative myota-contracts event documentation/schema and any required
owning-service helpers. Define the versioned envelope, stable message ID, subject
naming/registry, event versus work-command classification, consumer-group naming,
unknown-version behavior, payload bounds, correlation/causation fields, and backward
compatibility rules. Update synchronized myota-platform/myota-deploy contract copies
only through their documented sync process.

Make JetStream stream and durable-consumer provisioning deterministic and
drift-checked. Ensure every producer subject has required durable coverage before
publication under the selected retention policy. Prefer a single controlled
provisioner/reconciler over concurrent relay configuration mutation if that is what
the approved ADR specifies. Validate all correctness-sensitive consumer settings.
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
      Filter to supported event subjects where feasible; document explicit no-op
      dispositions rather than relying on broad catch-all consumers.
- [ ] Confirm Operations remains metadata-only and does not accidentally become a
      broker consumer with acknowledgements.

**Exit criteria**

- [ ] Inventory reconciliation finds no event write bypassing the outbox or
      unregistered event subject.
- [ ] Relay restart/retry proves no lost accepted event and deduplicates a
      publish/mark crash window.
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
publish through the registered envelope/subject path. Add or update explicit durable
consumers only for required domain-event groups in this phase; each must have clear
supported-type filters, idempotency, explicit ack-after-commit, version handling,
and a poison-event path. Do not add broad no-op consumers just to retain subjects
under Interest retention; update provisioning and retention decisions according to
the approved ADR.

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
      pattern. Preserve domain ownership: Activity can create notices from approved
      Identity/Geodata/Programme events, but must not own those domains' state.
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
      backlog disposition is understood.

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
own notification projections only. Preserve Operations as read-only status
inspection. Keep stable durable names, and document how to roll to a successor
durable without dropping Interest-retained events.

Update owning service code, container/deployment wiring, synchronized mirrors,
contracts, operations runbooks, and myota-docs. Add focused
delivery/restart/idempotency tests following existing conventions. Produce a
consumer-by-consumer matrix with evidence for supported versions, duplicate
delivery, poison event recovery, backlog metrics, and graceful shutdown. Mark only
verified rows complete.
```

### Phase 4 — Move Activity database-polled accepted work to JetStream

**Work**

- [ ] For each `activity_worker.py` job kind (`ADIF_IMPORT`, `QSO_INGESTION`,
      `AWARD_RECALCULATE`, `AWARD_EVALUATION`, `PDF_RENDER`, `STATISTICS_REBUILD`,
      `NOTIFICATION_SEND`), trace the producer, job row, side effects, retry
      semantics, payload size, idempotency key and completion state. Include
      `activity.adif.queued.v1` and related outbox writes.
- [ ] Move accepted jobs to transactional outbox + dedicated command subjects and
      durable pull queues, keeping the job/resource row as domain status and
      recovery evidence. Large payloads should remain in owned storage and the work
      event should carry identifiers, not copied content.
- [ ] Use per-kind or compatible worker-group durables and bounded concurrency; do
      not put unrelated long PDF/ADIF jobs behind a single serial queue unless
      ordering is explicitly required. Preserve leases for long-running work and
      heartbeat/visibility where appropriate.
- [ ] Use a dual-read/dual-publish migration only if Phase 0 approves it. Define
      event IDs and idempotency to prevent the same job running through both paths.
      Stop new DB-queue claims before draining old work; provide rollback without
      re-enqueueing completed jobs.
- [ ] Keep necessary periodic maintenance/reconciliation tasks in schedulers when
      they are timer-triggered rather than event-driven; record why they are not
      JetStream messages.

**Exit criteria**

- [ ] Every accepted Activity job is JetStream-backed or has a documented, approved
      exception; no job is acknowledged/completed before durable side effects.
- [ ] Cutover and rollback preserve exactly-once business effects under at-least-
      once delivery, despite both systems briefly seeing the same job.
- [ ] Activity job latency, queue age, failure, retry, and dead-letter states are
      visible in service and operations dashboards.

**ChatGPT prompt — Phase 4**

```text
Implement Phase 4: migrate the Activity service's accepted asynchronous jobs from
PostgreSQL polling to NATS JetStream work queues, following the approved Phase 0
decision and Phase 1 contracts. Inspect activity_repository.py, activity_worker.py,
outbox writes, job migrations/schema, Activity API job producers, deployment
manifests, and all recovery/retention paths before editing.

Inventory and handle these current job kinds individually: ADIF_IMPORT,
QSO_INGESTION, AWARD_RECALCULATE, AWARD_EVALUATION, PDF_RENDER, STATISTICS_REBUILD,
and NOTIFICATION_SEND. For each choose a versioned work subject and durable consumer
group, or document an approved exception. Preserve resource/job status in
myota_activity. Persist work request/outbox atomically with the accepted state
transition; publish identifiers and bounded metadata rather than large content. Keep
blob data in the configured object store. Ensure long-running tasks use suitable ack
wait/progress/lease strategy and independent concurrency so unrelated heavy work
does not block other kinds.

Implement at-least-once-safe processing: stable job/event ID, domain idempotency,
explicit ack after committed completion/failure state, bounded redelivery/backoff,
visible poison/dead-letter handling, graceful drain, and startup recovery. Design a
controlled cutover from DB polling, including old-job drain, duplicate prevention
during overlap, metrics, and rollback. Do not delete job history or claim exact-once
broker delivery. Keep scheduled retention/reconciliation tasks as schedules unless
the ADR specifically classifies them as event work.

Update Activity source, contract/subject registry, deployment and synchronized
myota-deploy/myota-platform copies, tests, Activity README and central MyOTA docs.
Validate each job kind with focused unit/integration evidence and report
cutover/rollback instructions, remaining DB polling, job age/backlog visibility, and
any blocked handler. Do not move Geodata work into Activity ownership.
```

### Phase 5 — Reconcile Geodata queues and cross-service recovery

**Work**

- [ ] Validate the four existing Geodata durables against the registered work
      contracts, shared provisioning, deployment replicas and Operations view.
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
      behavior; all four durables are validated in each environment.
- [ ] Broker loss, worker restart, delayed ack and duplicate delivery do not lose or
      repeat domain effects.
- [ ] Recovery loops are bounded, observable, and do not become a second primary
      dispatch path.

**ChatGPT prompt — Phase 5**

```text
Implement Phase 5: reconcile and qualify the existing Geodata JetStream work queues
and all cross-service recovery paths. The four current Geodata durable pull
consumers are preprocessing, import promotion, entity deletion, and location
enrichment. Read the event contract, Geodata architecture/runbooks, Phase 0
inventory, and approved ADR first.

Verify the producer transaction, work subject, provisioned durable/filter/config,
worker replica model, explicit ack policy, ack wait/max-deliver/max-ack-pending,
database idempotency/checkpoint/lease, long-work heartbeat, failure/dead-letter
behavior, and recovery source for each queue. Inspect transient failures and ensure
retryable lease contention is deferred rather than irreversibly terminated. Confirm
acknowledgement follows durable side effects. Preserve location request ID and
geometry-hash recheck, deletion authorization and Activity impact sequencing, import
cancellation semantics, and database reconciliation loops as repair mechanisms
rather than competing primary queues.

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
  consumer completion; restore broker and consumer state; demonstrate supported
  replay source for acknowledged work and show expiry recovery from database
  records/reconcilers.
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
  independent scaling. Test deployment ordering so consumers exist before
  producers publish under Interest retention.
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
