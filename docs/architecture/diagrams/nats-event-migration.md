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
  E --> N[Activity notifications<br/>broad durable]
  E --> GW[Four Geodata work durables]
  A -. DB polling .-> AW[Activity worker]
  G -. reconciliation .-> GR[Geodata recovery loops]
  E -. read-only metadata .-> O[Operations observer]
```

## Phase 2 relay source behavior — isolated verification

This records the relay implementation that passed isolated tests. The live
topology above was inspected separately; this flow does not claim a production
cutover.

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

**Status (10 October 2026):** Phases 0 and 1 contract/topology work and Phase 2
relay hardening/source coverage are complete within their evidence bounds.
The 68 fact schemas and ten selected work commands are registered; contract CI
checks their schema/disposition coverage and exact work-to-durable mapping. The
create-only provisioner and its drift checks passed focused and isolated tests.
The workspace owner accepted a cluster-internal NATS trust boundary without
authentication or TLS. The live NATS service is ClusterIP-only on port 4222;
the `myota` namespace has no NetworkPolicy, so any pod with network reachability
is trusted. Do not expose NATS outside the cluster. Provisional capacity,
restore/replay policy, and provisioner ownership are decisions, not production
qualification. The source relay now enforces registry routing, size bounds,
stable IDs, retries, dead-letter recovery, and metrics while validating legacy
topology read-only. The target streams and consumer/work paths have not been cut
over. Geodata payload minimization, schema/privacy enforcement, representative
full-window sizing, off-node/PVC-loss restore qualification, and production
compatibility transition remain open. See the recovery runbook and
[Phase 2 evidence](../../operations/messaging/evidence/phase2-relay-hardening-2026-10-10.md).
