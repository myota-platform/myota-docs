# NATS event and work topology

The diagrams distinguish observed live behavior from the selected target.
They describe ownership and delivery class; they are not evidence that the
target is deployed. See the [migration plan](../../operations/messaging/nats-event-migration-plan.md),
[joint review](../../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
[Phase 1 evidence](../../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md), and the [recovery runbook](../../operations/messaging/jetstream-recovery.md).

## Current deployed topology — observed 10 October 2026

```mermaid
flowchart LR
  subgraph DB[Service-owned PostgreSQL databases]
    C[Core outbox]
    A[Activity outbox and activity_job]
    G[Geodata outbox and recovery rows]
  end
  C --> RC[Core relay]
  A --> RA[Activity relay]
  G --> RG[Geodata relay]
  RC --> E[MYOTA_EVENTS<br/>file, Interest retention<br/>myota.events.* and myota.geodata.*]
  RA --> E
  RG --> E
  E --> N[Activity notifications<br/>activity-notifications-v1<br/>21 exact fact subjects]
  E --> GW[Four Geodata work durables]
  A -. DB polling .-> AW[Activity worker]
  G -. reconciliation .-> GR[Geodata recovery loops]
  E -. read-only metadata .-> O[Operations observer]
```

## Phase 2 relay behavior — deployed 10 October 2026

The relay mapping passed isolated checks and is now deployed to the local K3s
cluster. The stream and durable configuration remains the observed mixed legacy
topology above. This flow does not claim consumer or work-stream cutover.

```mermaid
flowchart LR
  subgraph DB[Owned transactional outboxes]
    C[Core]
    A[Activity]
    G[Geodata]
  end
  C --> R[Contract-backed relay<br/>one row at a time]
  A --> R
  G --> R
  R -->|registered facts<br/>stable Nats-Msg-Id<br/>size bounded| F[myota.events.&lt;dotted event type&gt;]
  R -->|six mapped source types<br/>legacy subject preserved| W[myota.geodata.* work]
  R -. topology checks only .-> E[Existing MYOTA_EVENTS<br/>Interest retention]
  R -->|retry, metrics, DB dead letter| D[Owning PostgreSQL]
  D -->|authorized, audited redrive| R
  O[Operations] -. metadata only .-> E
```

## Phase 4 Activity work — live in production

The production `MYOTA_ACTIVITY_WORK` stream uses WorkQueue retention and the
six exact pull durables. Two Activity workers subscribe to them; the obsolete
`activity_job_claim_idx` and synthetic `NOTIFICATION_SEND` jobs were retired by
migration 007. `activity_job` remains the status, history, and recovery source
of truth. The production backlog was zero at cutover, so live handler execution
is not claimed; isolated PostgreSQL/JetStream qualification processed all six
registered command types. See the [Phase 4 evidence](../../operations/messaging/evidence/phase4-activity-work-2026-10-10.md)
and [work queue runbook](../../operations/messaging/activity-work-queues.md).

## Selected target — ADR-0008, not yet live

```mermaid
flowchart LR
  subgraph DB[PostgreSQL remains authoritative]
    F[Domain mutations and outbox facts]
    AW[Activity state and work outbox]
    GW[Geodata state and work outbox]
  end
  F --> R[Database-specific outbox relays]
  AW --> R
  GW --> R
  R --> ES[MYOTA_EVENTS<br/>Limits, bounded 30-day fact window<br/>myota.events.&lt;dotted event type&gt;]
  ES --> CG[One durable per intended fact consumer group]
  R --> AS[MYOTA_ACTIVITY_WORK<br/>WorkQueue, bounded limits<br/>myota.work.activity.*]
  AS --> AC[Six independent Activity work durables]
  R --> GS[MYOTA_GEODATA_WORK<br/>WorkQueue, bounded limits<br/>myota.work.geodata.*]
  GS --> GC[Four independent Geodata work durables]
  AW -. scheduled/recovery only .-> AR[Activity maintenance/recovery]
  GW -. scheduled/recovery only .-> GR[Geodata reconciliation]
  ES -. inspect only .-> O[Operations observer]
```

**Status (10 October 2026):** Phases 0–4 are complete within their evidence
bounds. Activity work is deployed with two JetStream workers and six exact
durables. Migration 007 removed the claim index and synthetic notification
jobs while preserving Activity job history. Prometheus returns zero-valued
per-kind job metrics. Production had no selected work at cutover, so live
processing of a real Activity job is not claimed; isolated qualification
processed all six types. The mixed Interest-retained fact stream, the exact
Activity notification durable, and four Geodata work durables are unchanged.
Phase 5 Geodata work migration, payload privacy/schema enforcement, and the
final mixed-stream cutover remain separate gates. The accepted cluster-internal
trust boundary and deferred off-node recovery remain as recorded in ADR-0008.
See the [Phase 4 evidence](../../operations/messaging/evidence/phase4-activity-work-2026-10-10.md),
[Activity work queues runbook](../../operations/messaging/activity-work-queues.md),
and [Phase 3 evidence](../../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md).
