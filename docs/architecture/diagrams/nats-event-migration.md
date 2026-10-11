# NATS event and work topology

The diagrams distinguish historical rollout snapshots from the Phase 6
live topology. The lower current-state diagrams are grounded in the live
read-only K3s checks recorded in the
[Phase 6 evidence](../../operations/messaging/evidence/phase6-fact-stream-cutover-2026-10-11.md).
See the [migration plan](../../operations/messaging/nats-event-migration-plan.md),
[joint review](../../operations/messaging/evidence/phase1-joint-review-2026-10-10.md),
[Phase 1 evidence](../../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md), and the [recovery runbook](../../operations/messaging/jetstream-recovery.md).

## Current deployed topology — Phase 6 cutover, Helm 246

The direct cutover changed `MYOTA_EVENTS` to Limits retention and removed
the legacy Geodata subject capture after an empty-stream preflight. Helm
revision 246 is deployed with the one-time migration flag disabled. The MyOTA
Fleet bundle is Ready=True with 67/67 resources at Deploy main commit
`f7d2e65edd5248d70376cf99ccc98c836b7f95e3`. The four legacy Geodata durables
retired during Phase 5 remain absent. Operations no longer connects to NATS;
Surveyor owns broker monitoring.

```mermaid
flowchart LR
  subgraph DB[Service-owned PostgreSQL databases]
    C[Core outbox]
    A[Activity outbox and activity_job]
    G[Geodata outbox and recovery rows]
  end
  subgraph DEPLOY[Deploy owner: Helm 246]
    IMG[First-party images pinned by digest]
  end
  C --> RC[Core relay]
  A --> RA[Activity relay]
  G --> RG[Geodata relay]
  IMG -. deploys .-> RC
  IMG -. deploys .-> RA
  IMG -. deploys .-> RG
  RC --> E[MYOTA_EVENTS<br/>file, Limits, 30-day bounded facts<br/>myota.events.&gt; only]
  RA --> E
  RG --> E
  E --> N[Activity notifications<br/>activity-notifications-v1<br/>21 exact fact subjects]
  A --> AS[MYOTA_ACTIVITY_WORK<br/>WorkQueue, six exact durables]
  G --> GS[MYOTA_GEODATA_WORK<br/>WorkQueue, four exact durables]
  AS --> AW[Activity JetStream workers]
  GS --> GW[Geodata JetStream workers]
  A -. scheduled/recovery only .-> AR[Activity maintenance]
  G -. bounded row repair .-> GR[Geodata recovery loops]
  E -. broker metrics .-> S[NATS Surveyor]
```

## Historical Phase 2 relay behavior — 10 October 2026

This is a historical snapshot from Phase 2. At that point, facts still
published to the Interest-retained shared stream while mapped work commands
published to their disjoint WorkQueue streams. The live Phase 6 topology is
shown above; do not use the historical Interest-retention annotation as current
configuration.

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
  R -->|six mapped Geodata source types<br/>four bounded commands| W[myota.work.geodata.*]
  R -. fact validation only .-> E[MYOTA_EVENTS<br/>Interest retention]
  W --> G[MYOTA_GEODATA_WORK<br/>WorkQueue]
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

## Selected topology — now live after Phase 6

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
  ES -. broker metrics .-> S[NATS Surveyor]
```

**Historical Phase 5 snapshot (10 October 2026):** This view records Helm
revision 192 before the final Phase 5 durable retirement and Phase 6 cutover.
At that point, the legacy Geodata durables were awaiting retirement and
`MYOTA_EVENTS` still used Interest retention. Those states are superseded by
the current Phase 6 topology above. The accepted cluster-internal trust
boundary and deferred off-node recovery remain as recorded in ADR-0008. See the
[Phase 5 evidence](../../operations/messaging/evidence/phase5-geodata-work-2026-10-10.md),
[Phase 4 evidence](../../operations/messaging/evidence/phase4-activity-work-2026-10-10.md),
[Activity work queues runbook](../../operations/messaging/activity-work-queues.md),
and [Phase 3 evidence](../../operations/messaging/evidence/phase3-domain-consumers-2026-10-10.md).
