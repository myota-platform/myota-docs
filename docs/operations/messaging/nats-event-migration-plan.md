# NATS JetStream event and work-queue migration plan

**Status:** Phases 0–4 are complete within their recorded evidence bounds.
Phase 5's four Geodata work consumers, migration 021, retry-safe partial
deletion recovery, and Activity idempotency fix are live. Helm revision 192
completed at 21:39:22 UTC on 10 October; Fleet is Ready=True at Deploy commit
`6443473828305ab9d02a918bbe990d01abe97f6a` with 60/60 resources. Configured immutable image references match
live pod IDs. The four legacy Geodata durables remain empty and inactive through
the 24-hour observation ending no earlier than 21:39:22 UTC on 11 October 2026;
safe retirement remains open. No accepted production Geodata work was available
for processing. Phase 6 will handle the separate shared fact-stream retention
transition.

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
| Identity / `myota_core` | Account, callsign, role, authentication/recovery, OIDC mapping, and service-token lifecycle events. Examples: `identity.account.created.v1`, `identity.callsign.verified.v1`, `identity.account.deactivated.v1`. | Published by the shared `core-outbox` relay. | No Identity-owned JetStream consumer found. Activity's `activity-notifications-v1` durable filters the 19 Identity facts selected in the registry and creates only its local notice projection. Other facts have no selected subscriber. |
| Programme / `myota_core` | Programme create/update/archive, entity-type catalogue/assignment, content workflow, and policy-draft events. | Shared `core-outbox` relay. | No Programme-owned JetStream consumer found. Verify event subscribers and future ownership. |
| Activity / `myota_activity` | Activation/QSO lifecycle, ADIF queue, activity cascade deletion, award definition/request/issuance/rendering, and other activity/award events. | `activity-outbox` relay. | Activity notification group is `activity-notifications-v1` with exact registered filters; it does not subscribe to Activity facts. Migrate six accepted jobs (QSO ingestion, ADIF import, award recalculation/evaluation, PDF rendering, and statistics rebuild) to Activity work subjects. Exclude the state-only `NOTIFICATION_SEND` job; correct or remove it, and create a separate provider-backed command if external delivery is added later. |
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
      Production watermark comparison and controlled cutover remain rollout
      gates. Isolated relay retry/dead-letter qualification is recorded as
      complete in the Phase 2 evidence.
- [x] Add contract fixtures and CI checks for unique event/work subjects,
      per-event schemas, subscriber dispositions, source provenance, and
      registered work-to-durable topology. A new producer literal without a
      registry disposition fails the workspace source audit.

The Phase 1 registry and audit provide source coverage, but do not authorize
runtime publication.

#### Registry coverage and source audit

The Phase 1 registry is the contract inventory; Phase 2 added a generated
runtime routing catalog without making the registry a permanent event archive.

- **Coverage:** 68 domain facts and ten selected work commands are registered.
  Six current Geodata work/recovery source types map to four target commands.
- **Work contracts:** Each command has a bounded payload schema, source-linked
  producer and consumer paths, and an owning-row `workId` source.
- **Source audit:** `myota-contracts/contracts/event-registry.json` records the
  exact producer path for every fact. The workspace audit validates each source
  reference and classifies event-like Python literals as a registered fact or
  mapped legacy work type.
- **Result:** The 10 October audit found no undispositioned event-like literals
  across the five service repositories. Contracts CI passed the source audit
  on `main`.

#### Payload schema evidence and enforcement gates

- **Schema inventory:** Source-derived payload schemas cover all 68 facts:
  19 Identity, 12 Programme, 10 Activity, and 27 Geodata events. Identity,
  Programme, Activity, and Geodata schema checks passed in Contracts CI;
  recorded runs include [Geodata 38042564323](https://github.com/myota-platform/myota-contracts/actions/runs/38042564323)
  and [Activity 38040217352](https://github.com/myota-platform/myota-contracts/actions/runs/38040217352).
- **Relay enforcement:** Phase 2 source now caps the serialized message at
  1 MiB. It does not validate each payload against JSON Schema or enforce
  prohibited-field/purpose rules. Projection, privacy review, and consumer
  compatibility remain required before those checks can be enforced.
- **Geodata gate:** `geodata.import.preprocessed.v1` still carries the complete
  result object, including internal `_records` and `_status`, which can contain
  imported source features. Preserve v1 meaning and agree on a compatible
  successor projection before publishing minimized data.
- **Operations:** The Phase 0 audit found no Operations event-producing call
  sites. An Operations payload schema is not applicable unless it becomes a
  producer.
- **Limit:** Additive source-derived schemas describe observed shapes; they do
  not certify purpose limitation or retention.

#### Provisioner and CI evidence

- The deploy-owned provisioner fixes and validates pull delivery mode, explicit
  ACK, replay policy, retry limits, pending and waiting-pull bounds, consumer
  replicas, and full-payload delivery.
- Its topology source is synchronized to the platform mirror. Contracts CI
  checks out the five service repositories, runs the workspace event-source
  audit, regenerates the runtime routing catalogs, and compares synchronized
  runtime/migration copies.

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

- [x] Bring the shared relay to the contract: stable event IDs, bounded payloads,
      connection/reconnect behavior, bounded concurrency, retries/backoff,
      idempotent publish, publish-ack handling, failure metrics, and dead-letter
      inspection/replay. Ensure a crash after publish acknowledgement but before
      outbox marking safely republishes the same message ID. Unknown routes and
      serialized payloads above the configured cap dead-letter before publish.
- [x] Verify the core, activity, and geo relays see only their owned database and
      that retention cleanup cannot remove unpublished or unresolved dead-letter
      rows. Geodata import cleanup preserves unresolved redrive evidence. No
      indexes or partitioning were added without measured need.
- [x] Reconcile all registered event writes in Identity, Programme, Activity,
      Geodata, and Operations against the source audit. No undispositioned event
      literal or Operations producer was found. Selected asynchronous work remains
      on its documented owning-row path; no process-local cross-service dispatch
      was introduced.
- [x] Validate the existing Activity notification durable and four Geodata work
      durables against the live legacy stream without creating or changing broker
      topology. The broad Activity filter remains until its successor durable can
      be provisioned and drained under the Phase 3 consumer gate; event types with
      no intended subscriber remain explicitly listed in the registry.
- [x] Confirm Operations remains metadata-only and does not consume or acknowledge
      messages. Its broker inspection path uses read-only metadata APIs.

**Exit criteria**

- [x] Inventory reconciliation finds no unregistered event source or subject;
      source-linked producer records and helper transactions cover the registered
      event set.
- [x] Relay restart/retry proves no lost accepted event and safely recovers from a
      publish/mark crash: broker deduplication applies within its configured window,
      and database idempotency protects retries outside that window.
- [x] Outbox backlog, oldest age, retries, and dead letters are observable and
      actionable for all three relays through per-database metrics, Collector scrape
      targets, dashboards, alerts, and the database-authoritative redrive CLI.

**Phase 2 implementation evidence:** see the
[relay hardening evidence record](evidence/phase2-relay-hardening-2026-10-10.md).
The relay validates the existing mixed Interest stream and durable settings
read-only; it does not provision, update, or delete broker topology. The live
legacy filter set was inspected read-only on 10 October. Service-side outbox
write helpers and consumer handlers were not changed. The relay now routes
registered facts to dotted contract subjects; the six mapped Geodata work
source types retain their current legacy subjects. The Activity notification
durable remains broad until its registered-filter successor can be introduced
without abandoning its current Interest-retained backlog. The target
Limits/work-stream cutover remains gated.

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

- [x] Standardize Activity notification and all other event consumers around a
      shared service-owned JetStream adapter or an explicitly documented per service
      pattern. Preserve domain ownership: Activity notification filters cover the
      approved Identity facts and the two Geodata review/status facts in the
      inventory. Do not add Programme notices or other subscriptions without an
      owner and registered consumer-group decision.
- [x] Separate independent consumers into separate durables. Set explicit filter,
      ack wait, max deliveries, max ack pending, backoff, delivery policy, and
      concurrency based on measured handler duration and recovery needs.
- [x] Commit domain side effect and consumer deduplication/checkpoint in one local
      database transaction when possible; acknowledge only after commit. Validate
      behavior when the database commit succeeds but ack is lost.
- [x] Define poison-message handling that records a redacted diagnostic envelope safely,
      avoids exposing secrets/PII in logs, notifies operations, and supports
      reviewed replay after remediation. Avoid immediately terminating errors
      without a supported recovery path.
- [x] Keep durable consumer names stable across releases; create successor durables
      deliberately and remove obsolete durables only after old workers drain and
      backlog disposition is understood. For fact replay, use a separate durable
      with an explicit start position; never rewind a production side-effecting
      durable. Preserve the selected 30-day bounded fact replay window and use
      database state as the source for longer-term reconstruction.

**Exit criteria**

- [x] Every intended domain-event consumer group is live, documented, independently
      deployable, idempotent, observable, and tested against redelivery/restart.
- [x] No unsupported broad consumer determines retention accidentally.

**Phase 3 implementation evidence:** see the
[domain-consumer evidence record](evidence/phase3-domain-consumers-2026-10-10.md)
and the [Activity notification runbook](activity-notification-consumer.md).
The only selected domain-event group is Activity notifications. Its exact-filter
successor is provisioned before worker rollout; the two known broad Activity
durables are retired after successor validation. The mixed Interest-retained
stream and all work-command paths are unchanged by this phase.

**ChatGPT prompt — Phase 3**

```text
Implement Phase 3 of the MyOTA NATS migration. Read this plan, the Phase 0
inventory, ADR-0008, and the Phase 1/2 evidence before editing. Work in the
authoritative repositories `myota-activity-service`, `myota-deploy`,
`myota-contracts`, and `myota-docs`. Keep `myota-platform` and any deploy-side
service copies synchronized mirrors; keep `.github` links to current docs valid.

Scope is domain-event subscribers only. The contracts registry defines the
intended subscriber groups. Currently this is Activity notifications for the
approved Identity facts and the two Geodata review/status facts. Do not add
Programme subscriptions, change producer/outbox paths, migrate Geodata work, or
migrate Activity database-polled jobs. Preserve ownership: Activity may write
only its local notification projection; Identity and Geodata remain owners of
their records. Operations remains read-only for broker inspection.

For every intended group, set an exact registered filter and a stable durable.
Use pull delivery, explicit ACK, bounded ack wait/max deliveries/max ack pending,
bounded retry/backoff, and same-durable replicas. Commit side effects and
consumer deduplication/checkpoint state before ACK. Where the business side
effect has a separate transaction, require a stable unique idempotency key and
prove commit-success/ack-loss does not duplicate it. Avoid storing source payloads
in projections unless the contract requires them. Redact secrets and personal
data from diagnostics and logs.

Create successor durables before moving workers. Retire an obsolete durable only
after validating its successor and establishing the disposition of its matching
backlog. Do not rewind a production durable. Keep the existing mixed
Interest-retained `MYOTA_EVENTS` stream unchanged in this phase; do not claim
that it is the selected Limits-retained target. Provide a reviewed poison-event
recovery path with a durable application record, operator audit, and stable
event identity. Keep JetStream backlog and consumer outcome metrics bounded in
cardinality and visible to Operations.

Update owning service code, deployment hooks/wiring, contracts checks, mirrors,
runbooks, diagrams, backlog, and the implementation timeline. Add focused
redelivery, restart, poison/redrive, and shutdown checks; run broker/database
integration checks in a disposable namespace on this host and remove all test
resources afterward. Report exact commands/results, live durable filters and
pending state, changed repos/files, and remaining gates. Mark work and exit
criteria complete only after the deployed group is verified and no unsupported
broad durable still influences retention. Commit and push changes directly to
`main` with explicit messages.
```

### Phase 4 — Move Activity database-polled accepted work to JetStream

**Status:** complete within the production workload evidence bound. The
production cutover, migration, cleanup, and live readiness checks are recorded
in the [per-kind migration evidence](evidence/phase4-activity-work-2026-10-10.md)
and [Activity work queue runbook](activity-work-queues.md). No selected job
was queued during cutover; therefore the production stream is empty and no
production handler execution is claimed. Isolated end-to-end qualification
processed all six registered command types.

**Work**

- [x] Trace each selected job kind to its producer, owning-row transaction,
      source payload, side effects, idempotency boundary, completion state, and
      retry behavior. The six command routes and exact source paths are recorded
      in the Phase 4 evidence table. ADIF upload bytes and PDF objects remain in
      object storage; command payloads contain only `jobId`.
- [x] Keep `NOTIFICATION_SEND` out of JetStream. Activity inserts an in-app
      notification as `DELIVERED` in its owner transaction. The migration
      corrects any legacy queued notification state and deletes only synthetic
      `NOTIFICATION_SEND` job rows.
- [x] Add atomic job/outbox writes and route the six accepted job kinds to
      registered `myota.work.activity.*` subjects. The registry carries strict
      `{jobId}` payload schemas, a stable job UUID work ID, `activity_job`
      aggregate identity, and the `activity-service` producer.
- [x] Implement six independent pull durables with explicit ACK, exact filters,
      bounded pending/concurrency, ack progress, retry/backoff, lease heartbeat
      and UUID fencing, terminal work-DLQ persistence, and audited DB redrive.
      The worker fails closed on missing or drifted pre-provisioned durables.
- [x] Add migration `007_activity_jetstream_work.sql` to backfill queued work,
      preserve `available_at`, guard against first-run cutover with legacy
      `RUNNING` jobs, and add lease/DLQ/audit schema. Isolated PostgreSQL checks
      passed both the first-run gate and repeated migration while a JetStream
      job was running.
- [x] Retire the obsolete DB-poller index in schema source and migration:
      remove its recreation from `001_activity_relational.sql`, drop
      `activity_job_claim_idx`, and add `activity_job_status_kind_idx`. Preserve
      `activity_job` for API status/history/idempotency and preserve import,
      issuance, and domain history. No unrelated domain table is deleted.
- [x] Provision and validate only `MYOTA_ACTIVITY_WORK` and its six exact
      durables in a disposable host-K3s namespace. A second provision was
      idempotent; real JetStream/PostgreSQL duplicate-safe ACK checks returned
      pending and stream message counts to zero. The namespace was deleted.
- [x] Keep the ADIF retention CronJob and other timer/reconciliation paths on
      their owning scheduled/database recovery boundary. They are not normal
      accepted-work dispatch.
- [x] Execute the ordered production cutover: provision the disjoint target,
      scale the old Activity worker to zero and verify its pods stopped, run the
      guarded backfill, then roll out the new Activity API/worker images. Do not
      change the mixed `MYOTA_EVENTS` stream or Geodata work durables. The
      migration succeeded at Helm revision 176; the current chart rollout is
      revision 181 and uses the pinned Activity image.
- [x] After all old Activity API pods are gone, confirm the compatibility
      outbox-repair loop reports zero repairs and disable that transitional
      database scan. Verify no worker job-claim polling code or obsolete
      database claim index remains in the deployed system. The deployed value
      is `legacyReconciliationEnabled: false`, and the old claim index is
      absent.
- [x] Verify live per-kind job status/age/latency/retry/DLQ metrics and the
      Operations dashboard after rollout; record exact durable pending,
      ack-pending, and redelivery state. Activity exports zero-valued per-kind
      series; Prometheus returns the job metric; the Operations dashboard and
      alert rules query the same metric names. No latency observation exists
      because no production job ran.

**Exit criteria**

- [x] All six selected Activity kinds have registered routes, provisioned
      exact durables, and live pull workers. Production had zero selected jobs
      at cutover (so no live message execution is asserted); isolated
      PostgreSQL/JetStream qualification processed one command per durable.
      `NOTIFICATION_SEND` remains delivered in-app state and no synthetic job
      rows remain.
- [x] No accepted job is completed or ACKed before its committed side effects
      and job state. At-least-once redelivery, stale lease fencing, bounded
      retries, terminal DLQ, and audited redrive preserve idempotency.
- [x] The old DB claim index and DB polling worker are retired after the
      successful cutover; API job status/history and the Activity database remain
      authoritative. Scheduled retention/reconciliation remains on its recorded
      scheduler/recovery path.
- [x] Activity job latency, queued age, failures, retries, and unresolved
      dead-letter state are exposed to Prometheus and represented in the
      Operations dashboard/alerts. Latency remains unobserved until real work
      completes; no synthetic production command was inserted to fabricate it.
- [x] The staged rollout and job/outbox reconciliation passed without changing
      Geodata routes or mixed-stream retention. The forward-compatible rollback
      procedure is documented and reviewed against the zero-backlog cutover:
      stop JetStream workers, restore the prior API/DB-poller image, leave
      migration 007's lease/DLQ schema and purged synthetic notification rows
      in place, then verify job state. No production app rollback was needed or
      executed. The old claim index is a performance aid, not a correctness
      dependency; never reverse the committed migration or delete recovery
      evidence.

**ChatGPT prompt — Phase 4**

```text
Implement Phase 4 only: move Activity's six accepted database-polled jobs to
`MYOTA_ACTIVITY_WORK`, then complete and verify the production cutover. Read
ADR-0008, the Phase 0 inventory, Phase 1/2/3 evidence, this plan, and AGENTS.md
first. Authoritative sources are `myota-activity-service` for Activity job,
consumer, and migration code; `myota-contracts` for registered schemas and work
routes; `myota-deploy` for relay, topology, Helm, and operations wiring; and
`myota-docs` for evidence and decisions. Keep `myota-platform` service,
migration, deployment, registry, and observability copies synchronized. Keep
`.github` links to the current docs valid.

Migrate only `QSO_INGESTION`, `ADIF_IMPORT`, `AWARD_RECALCULATE`,
`AWARD_EVALUATION`, `PDF_RENDER`, and `STATISTICS_REBUILD`. Read each API
producer, job-row transaction, side-effect handler, idempotency key, retry path,
status endpoint, retention path, and source-specific payload before editing.
Use one atomic owner-database write for accepted state/job/outbox. Use stable
UUID job/work identity; send only bounded row identifiers through JetStream.
Keep ADIF files and rendered PDFs in their configured object storage. Preserve
Activity API job status/history and database authority. Keep at-least-once
processing, idempotent side effects, explicit ACK after committed completion
or terminal state, bounded concurrency/backoff, renewable leases with stale
worker fencing, persisted terminal failures, and authorized/audited database
redrive. Do not claim exactly-once delivery or publish directly from a worker.

Do not migrate state-only `NOTIFICATION_SEND`; its in-app notification is
already delivered and has no external provider side effect. Preserve existing
scheduled ADIF retention and other timer/reconciliation recovery paths. Keep
Geodata work, Operations' read-only broker inspection, the shared mixed
Interest-retained `MYOTA_EVENTS` stream, and all Geodata legacy durables
unchanged. Provision only the disjoint Activity WorkQueue stream and its six
registered pull durables; require exact drift validation and fail closed if
any required durable is absent.

Use an ordered rollout so the old database poller cannot race the backfill:
provision and validate the target; scale the old Activity worker Deployment to
zero and wait until its pods are gone; verify no selected job is still RUNNING;
run the idempotent migration/backfill; roll out the new Activity API and worker
images; verify every selected queued job has an outbox row; and retain the
compatibility repair only until old API replicas are gone, then disable it.
Rollback may stop new workers and return them to zero, but must not re-enqueue
completed work or restore the obsolete polling index. Preserve all job/domain
history; remove only schema objects and synthetic rows that exist solely for
the retired polling/state-only command after their replacement is verified.

Test focused producer, route, ACK/lost-ACK, duplicate, retry, crash/lease,
DLQ/redrive, idempotency, migration rerun, and cleanup behavior. Run broker and
database integration checks only in a disposable namespace on this host and
remove test data/resources afterward. Deploy through Helm/Fleet on the local
K3s cluster and verify the actual migration, six live filters/durables, job
metrics/dashboard, zero legacy job claims, and unchanged Geodata/shared fact
stream. Update the plan, evidence, work queue runbook, diagram, history with
this exact prompt, changes, README links, To do, Work in progress, prioritized
backlog, and synchronized mirrors. Check boxes only when evidence verifies
them. Commit and push all affected repository work directly to `main` with
explicit messages. Report changed paths/repositories, exact checks and live
results, remaining risk, schema objects retired, namespace cleanup, and each
Phase 4 exit criterion.
```

### Phase 5 — Reconcile Geodata queues and cross-service recovery

**Status (10 October 2026):** Production routes all four Geodata work
kinds to the immutable image-referenced `MYOTA_GEODATA_WORK` WorkQueue. Helm revision 192 completed at 21:39:22 UTC on 10 October; Fleet reports
`Ready=True` at Deploy commit `6443473828305ab9d02a918bbe990d01abe97f6a`, with 60/60 resources ready. This is
the current rollout and observation anchor; configured image digests are
unchanged from revision 191. The Deploy image
references and live Activity/Geodata/outbox pod IDs resolve to the configured
immutable digests. Activity runs
`sha256:f764bfe7193ba8166c84f3d6b063547f94fcc17b6c819aa597c617ed6c835261`;
Geodata runs
`sha256:c6f5dee746579469ba827a4af78741acbe5d25e825471c84c95ec5574e6e2aee`;
the shared runtime runs
`sha256:1f4002619cee64d9d05f06b96c93d348df5ae725806d08da383d74e0c84c91e8`.
The Activity idempotency fix is now live. The target stream and all four
replacement durables are healthy and empty. The four prior Geodata durables in
`MYOTA_EVENTS` are empty and inactive.

The rollout check found that recorded digests previously changed only pod
annotations while containers still referenced mutable `:latest` tags. Helm
now renders first-party runtime, worker, provisioning, and scheduled-job
images as `repository@sha256:…` whenever the digest is configured. Both Deploy
and Platform carry the fix, and live pod image references match their
configured digests.

The current 24-hour rollback observation starts from revision 192's completed
rollout at 21:39:22 UTC on 10 October. Keep the four legacy Geodata durables
through 21:39:22 UTC on 11 October 2026; then recheck Fleet, migration markers, recovery age and all
legacy backlog counters before retiring only those four durables. Do not remove
`MYOTA_EVENTS`, Activity's notification durable, Geodata source tables, work
rows, outbox or recovery columns/indexes. No accepted production Geodata work
was available to exercise.

**Work**

- [x] Move preprocessing, promotion, confirmed deletion, and location enrichment
      from their legacy `MYOTA_EVENTS` subjects to the four disjoint
      `myota.work.geodata.*` subjects in `MYOTA_GEODATA_WORK`. Provisioned the
      target before cutover. Production workers now subscribe only to the new
      subjects; the old durable definitions are retained inactive for rollback.
- [x] Validate the four replacement pull durables against the registered work
      contracts, exact filters, explicit ACK, max-delivery, pending bounds,
      stream limits and deployment-owned provisioner in the isolated and live
      K3s environments.
- [x] Verify the committed work source, stable owner-row `workId`, duplicate
      boundary, heartbeat, lease and ACK/NAK behavior. A refused database
      connection caused NAK/redelivery and settled after reconnect without a
      false ACK. A partial Activity/Geodata deletion failure now remains
      retryable rather than being marked terminal and ACKed.
- [x] Keep stale cancellation and pending-deletion reconcilers as age-bounded,
      batch-limited repair paths. Database-backed tests recreated each of the
      four work kinds once through the outbox; the next scan emitted none.
      Location enrichment rechecks request ID and geometry hash before applying
      provider results.
- [x] Verify Activity cascade sequencing across two disposable service
      databases: Activity success followed by an injected Geodata-side failure
      NAKed the JetStream delivery; the retry completed the job, deleted the
      Geodata entity, emitted exactly one Activity cascade outbox fact, and
      drained the private stream. Activity persists the request idempotency key
      transactionally and serializes concurrent requests with a database
      advisory lock. Its immutable image digest is deployed to the Activity
      API, workers and notification consumer; Helm 192/Fleet readiness is
      verified.
- [x] Qualify cancellation racing with an acknowledged Geodata preprocessing
      delivery. The worker claimed the run, the cancellation API committed
      CANCELLING, and the same delivery finalized CANCELLED before ACK; the
      durable settled with zero messages/pending/ack-pending. This matches the
      current boundary: imports are cancellable; confirmed deletion jobs are not.
- [x] Connect message expiry to owner-row recovery and completion. An original
      disposable deletion message expired, the age-bounded scanner reconstructed
      a command through the Geodata outbox, and JetStream redelivery completed
      the job with one Activity cascade fact and no remaining message.
- [x] Prove accepted work can be reconstructed from the owner row/outbox within
      the recorded age bound. Same-node NATS Pod restart with its persistent PVC
      retained an unacked message and durable state. A connected expiry →
      owner-row/outbox reconstruction → JetStream redelivery → completion check
      also passed. Off-node restore remains deferred.
- [x] Recheck Fleet and production topology/schema after immutable image
      rollout. Helm revision 192 is deployed; Fleet reports Ready=True at Deploy
      commit `6443473828305ab9d02a918bbe990d01abe97f6a` with 60/60 resources ready. Activity, Geodata and
      shared-runtime pod image references resolve to configured immutable
      digests. The target durables and schema markers are healthy. Keep
      Operations read-only.

**Exit criteria**

- [x] The four Geodata queues have documented scaling, retry, DLQ,
      recovery and retention behavior; replacement durables are validated in
      live and disposable environments. No Phase 5 database object is obsolete:
      migration 021's dispatch columns/indexes and domain/outbox rows remain
      required recovery state.
- [ ] Complete the 24-hour rollback observation after immutable-image Helm
      revision 192, then recheck replacement/legacy filters, backlog counters,
      migration markers, owner-row recovery age and Fleet readiness. Retire
      only the four legacy Geodata durable definitions after this check. The
      earliest time is 21:39:22 UTC on 11 October 2026.
- [x] Broker Pod restart, worker restart, delayed ACK, duplicate/lost ACK,
      expiry/reconstruction/completion, database failure, the Activity-success/
      Geodata-failure retry, and cancellation racing with an acknowledged
      preprocessing delivery preserve durable effects in isolated tests.
- [x] Build and publish the Activity cascade idempotency change,
      configure its immutable image digest, deploy it through Fleet, and
      recheck live health and topology. The Activity API, workers and
      notification consumer use the configured digest; Helm revision 192 and
      Fleet readiness are verified.
- [x] Recovery loops are observable, age-bounded and limited to batches of 50;
      they enqueue repair through the transactional outbox and do not run as a
      second primary dispatcher.

**ChatGPT prompt — Phase 5**

```text
Continue work implementing phase 5 iteratively,, same criteria and instructions as last phase.

Use the Phase 5 tasks and exit criteria in this plan as scope. Read ADR-0008,
the Phase 0 inventory, relevant Geodata and Activity runbooks, and the Phase 5
evidence before changing code. Treat myota-geodata-service as authoritative for
Geodata domain handlers and migrations; myota-deploy owns Helm, relays and
deployment migrations; myota-platform mirrors deployment/runtime files;
myota-contracts owns registered schemas. Keep PostgreSQL as the source of truth
and Operations read-only.

Do not dual-publish. Keep scheduled maintenance on its scheduler and stale-row
scans as bounded recovery only. Preserve confirmed-deletion authorization,
Activity impact/cascade semantics, cancellation, location request/geometry
checks, idempotent work IDs, explicit ACK after durable effects, and bounded
retry/dead-letter behavior. Keep the legacy durable definitions until the
recorded rollback observation expires and database recovery is rechecked.
Do not purge authoritative rows, outbox history, dead letters, or migration
021 objects that remain in use.

Qualify database failure, duplicate/lost ACK, delayed handler, worker/broker
restart, message expiry/reconstruction, cancellation race, and partial
Activity/Geodata deletion using disposable data in a separate host-K3s
namespace; clean up all test resources afterward. Never inject test data into
production. Commit and push each owning repository directly to main, keep
deployment mirrors synchronized, and update the phase evidence, timeline
(prompt used), diagrams, README links, todo, work-in-progress, prioritized
backlog, changes, and .github links.

Mark checkboxes only after exact evidence verifies them. Continue until each
exit criterion is met; if a safe time-based rollback gate remains, record its
earliest completion time and keep Phase 5 open. Report repositories and commit
SHAs, checks/results, live route and image/schema evidence, objects retained or
retired, cleanup, and any remaining evidence gap.
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
