# NATS migration Phase 1 contract and topology evidence

**Status:** Phase 1 is in progress. Contract and create-only provisioning
preparation is checked locally; no live stream, producer, or consumer behavior
was changed. This record does not satisfy the Phase 1 exit criteria.

## Work completed in this pass

- `myota-contracts/contracts/event-registry.json` now lists 68 domain facts,
  maps the six legacy Geodata work event types to four proposed commands, and
  defines six proposed Activity commands. The event registry contains 68
  per-event JSON Schema wrappers and shared event/work envelope schemas.
- `myota-contracts/scripts/generate_event_registry.py` generates the schema
  wrappers from the contracts-owned registry; `sync_contract_mirrors.py` now
  copies the event docs, registry, and schemas to the integration mirror.
- `myota-deploy/services/jetstream_topology.py` defines the three ADR-0008
  streams and ten work durables. Finite byte/message limits are mandatory
  inputs; no unmeasured production capacity defaults are invented.
- `myota-deploy/services/provision_jetstream.py` is create-only and validates
  stream limits and consumer delivery configuration. It fails on drift and
  cannot alter or delete existing streams/consumers. A Compose profile exposes
  it only through explicit operator invocation.
- The schemas currently constrain envelope fields and event identity. Event
  payload shapes remain explicitly marked pending owner evidence, so the
  registry is not yet a complete producer compatibility gate.

## Verification

| Check | Result |
|---|---|
| Contract registry/schema unit tests | 2 passed |
| Topology validation unit tests | 3 passed |
| Python compile check for topology/provisioner | Passed |
| Temporary isolated JetStream broker | Provisioned all 3 streams and 10 durables; an identical second run passed |
| Drift rejection | Changed the test broker's expected event-stream byte cap; provisioning failed with `configuration drift (max_bytes)`, then passed again with the original value |
| Production broker mutation or test publish | Not run; the shared production stream has legacy work and remains Interest-retained |
| Application/domain test data created | None; temporary broker state was removed with the test namespace |

The deployed service was inspected read-only through the existing core outbox
pod on 9 October 2026. It reports one stream, `MYOTA_EVENTS`, with subjects
`myota.events.>` and `myota.geodata.>`, Interest retention, file storage, one
replica, 30-day max age, and unlimited bytes/messages/message size. At inspection
it had zero messages, zero bytes, and the five existing durables had zero
pending and ack-pending messages. The NATS PVC requests 8 GiB. The chart starts
NATS with `-js -sd /data` and does not mount a server auth configuration;
per-role least privilege is therefore an open deployment gate.

Read-only aggregate queries against the three outbox databases found no
unpublished rows. Counts and payload sizes were:

| Database | Rows in the query's 30-day window | Observed row date span | Total payload bytes | Largest payload |
|---|---:|---|---:|---:|
| `myota_core` | 162 | 2–9 October | 15,860 | 387 bytes |
| `myota_activity` | 3,016 | 6–8 October | 452,400 | 150 bytes |
| `myota_geo` | 15,925 | 6–9 October | 8,889,104 | 373,607 bytes |

These rows do not cover a full 30-day operating window. The Geodata sample
includes load-test activity and an unusually large payload. The figures are
useful observations, not a defensible 30-day forecast or WorkQueue capacity
model. The observed largest payload also reinforces the requirement that
commands reference stored data instead of copying import content.

## Remaining Phase 1 gates

- Owning services must review and complete payload schemas, compatibility
  policy, event dispositions, and trusted correlation/causation propagation.
- Measure a representative operating window, classify normal versus test
  traffic, and choose finite stream limits against the 8 GiB single-node PVC,
  recovery objectives, and operational reserve.
- Add authenticated NATS configuration and least-privilege relay/worker
  credentials before exposing a provisioner to production.
- The existing relay still creates legacy durables and changes retention during
  startup. Phase 2 must remove that mutation and require pre-provisioned,
  validated topology before the provisioner can be activated.
- Exercise backup/restore, stream and durable recreation, disk-pressure
  backpressure, and bounded fact replay against an isolated broker. The local
  executor has no Docker CLI, and the shared live broker was not modified.

The isolated test ran in a temporary namespace on the host's K3s cluster. It
used an emptyDir-backed NATS pod and the already-deployed application image with
the working-tree provisioner source mounted read-only. It created no domain
events or application data. The temporary namespace and its resources were
deleted and verified absent after the test. Docker is not installed in the
executor, so Compose-based tests were unavailable. Test-only finite values were
8 MiB/10,000 messages/1 MiB per event for facts and 4 MiB/1,000 messages/1 MiB
per command for each work stream; Activity and Geodata test max ages were seven
and 30 days. These are harness inputs only, not selected production limits.

The exact Phase 1 prompt used is the copyable
[Phase 1 prompt in the migration plan](../nats-event-migration-plan.md#chatgpt-prompt--phase-1).
