# Phase 1 completion evidence — 10 October 2026

**Scope:** contracts, selected topology, create-only provisioning, and bounded
local restore/replay qualification. No runtime producer or consumer path and no
resource in the production `myota` namespace was changed.

## Decisions applied

- NATS remains cluster-internal at its `ClusterIP` service on the current
  single-tenant K3s cluster. NATS authentication and TLS are not Phase 1
  requirements. Any pod with network reachability can connect; keep NATS
  unexposed and revisit if the trust boundary changes.
- The selected finite stream caps are 1 GiB for facts, 1 GiB for Activity work,
  and 3 GiB for Geodata work, with at least 3 GiB of the current 8 GiB PVC
  reserved. Each stream has a 30-day age limit, finite message count, 1 MiB
  maximum message size, file storage, one replica, and `DiscardNew`.
- The 30-day database query returned less than eight days of rows and included
  Geodata load-test traffic. Volker accepts this evidence limitation for the
  current single-node scope. Keep the proposed limits out of the live mixed
  stream until the later compatibility cutover, monitor disk/outbox pressure,
  and remeasure a representative 30-day window before increasing a cap.
- PostgreSQL remains authoritative for business state and work recovery.
  Off-node backup is deferred and is not a Phase 1 gate. The accepted residual
  risk is loss of the bounded JetStream transport window if the node/cluster is
  lost; reconcile or redrive work from its owning database.

## Verification on local K3s

All broker tests ran in the temporary namespace
`nats-phase1-verify-20261010`, using a 1 GiB disposable PVC and the already
available `nats:2.10-alpine` image. The production `myota` namespace was not
modified.

| Check | Result |
|---|---|
| Create-only provisioner initial run | Created/validated all three selected streams and all ten registered work durables. |
| Idempotent repeat | Second run validated the same topology with no updates. |
| Drift rejection | A changed `MYOTA_EVENTS` byte cap failed with `configuration drift (max_bytes)`; the stream remained at 1 GiB. |
| `DiscardNew` capacity behavior | A disposable 256-byte stream accepted the first payload, rejected the next with JetStream `503 / maximum bytes exceeded`, and retained exactly one message. |
| PVC restore | Stopped the broker, copied its local PVC data to a new disposable PVC, and restarted NATS from that PVC. NATS restored `MYOTA_EVENTS` with 3 messages and recovered the provisioned durables. |
| Bounded replay | A new explicit durable replayed synthetic sequences `[0, 1, 2]` from the restored Limits stream. The test durable and all synthetic messages existed only in the temporary namespace. |
| Source/contract checks | 68 fact registry entries and ten per-command work schemas reconciled with authoritative service source; no undispositioned event-like source literals. Contracts unit tests: 4 passed. |
| Deploy checks | Full deploy test suite: 36 passed, 1 skipped because Pillow is not installed in the host test environment. Topology focused tests: 4 passed. |
| Helm barrier | Default and Spainip chart lint passed. The opt-in pre-upgrade Job rendered with explicit capacity/gate values; rendering with only `enabled=true` failed closed. |
| Mirror checks | Contract registry, event/work schemas, contract docs, provisioner, topology, chart values/template, and topology guide match their platform mirrors. |

The local PVC copy/restore qualifies same-node transport recovery only. It does
not qualify node-loss or cluster-loss recovery and is not described as an
off-node backup.

## Phase boundary

Phase 1 completes the versioned envelope and subject/schema registry, finite
target topology, create-only provisioner, and local restore/replay runbook.
Producer enforcement, payload minimization (including the Geodata preprocessed
event projection), relay-side unknown-route behavior, removal of legacy relay
topology mutation, and production cutover remain Phase 2 gates. The live shared
`MYOTA_EVENTS` stream remains mixed under Interest retention until that migration
barrier; the target provisioner must not be run against it.

## Exit criteria

- [x] Contract registry covers 68 committed facts and ten selected work commands;
      per-work payload schemas reference bounded database IDs, and subject
      namespaces and durable filters are non-overlapping.
- [x] Create-only topology provisioning is idempotent and rejects drift on local
      K3s; finite limits and `DiscardNew` behavior are verified in isolation.
- [x] Local disposable-PVC restore recovers stream data and durable state; replay
      through a new durable is verified.
- [x] Recovery authority, accepted capacity evidence limitation, and deferred
      off-node recovery risk are recorded.
- [x] NATS remains internal without auth/TLS by explicit decision; the accepted
      reachability risk and revisit triggers are recorded.
- [x] Production topology and runtime producer/consumer paths remain unchanged;
      Phase 2 gates remain open and are not represented as completed.
