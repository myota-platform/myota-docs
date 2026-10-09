# NATS event and work topology

The diagrams distinguish observed live behavior from the selected target.
They describe ownership and delivery class; they are not evidence that the
target is deployed. See the [migration plan](../../operations/messaging/nats-event-migration-plan.md)
and [Phase 1 evidence](../../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md).

## Current deployed topology — observed 9 October 2026

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
  E --> N[Activity notifications<br/>broad durable]
  E --> GW[Four Geodata work durables]
  A -. DB polling .-> AW[Activity worker]
  G -. reconciliation .-> GR[Geodata recovery loops]
  E -. read-only metadata .-> O[Operations observer]
```

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

**Status:** Phase 0 is complete. Phase 1 contract and create-only provisioning
preparation is in progress. No target stream or runtime path has been changed.
