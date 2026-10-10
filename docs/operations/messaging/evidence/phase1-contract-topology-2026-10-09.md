# NATS migration Phase 1 contract and topology evidence

**Status:** Phase 1 is in progress. Contract and create-only provisioning
preparation is checked locally; no live stream, producer, or consumer behavior
was changed. This record does not satisfy the Phase 1 exit criteria.

## Work completed in this pass

The implementation artifacts were merged: [contracts #2](https://github.com/myota-platform/myota-contracts/pull/2),
[deploy #4](https://github.com/myota-platform/myota-deploy/pull/4),
[platform mirror #1](https://github.com/myota-platform/myota-platform/pull/1),
and [organization profile #1](https://github.com/myota-platform/.github/pull/1).
The merge commits are contracts `1ed27b6`, deploy `b1038e6`, platform `38f0d69`,
and organization profile `a3cbc2c`. Merge closes review status; it is not production
approval. CI results are recorded in the verification table below.

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
| Contracts CI | Contract freeze check, Ruff formatting, and Ruff lint passed after formatting correction |
| Deploy CI | Ruff formatting and lint passed after formatting correction |
| Platform CI | Unit-test job and Ruff formatting/lint passed; container job could not start because Docker Hub token requests timed out twice |
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

## 10 October 2026 follow-up

The deploy-owned provisioner now pins and checks correctness-sensitive durable
consumer settings: explicit ACK, all-message/instant replay, bounded pending and
waiting pulls, redelivery count and timeout, inherited stream replicas, durable
file-backed consumer state, and full payload delivery. Drift in waiting-pull or
header-only delivery is rejected. These settings were copied to the platform
integration mirror. Four focused tests passed in both deploy and platform. Ruff format check and lint
passed in both repositories; the deploy-owned files and documentation compare
byte-for-byte with the platform mirror. A disposable `nats:2.10-alpine` broker in a
temporary host-K3s namespace accepted initial provisioning and an idempotent second
run for all three streams and ten durables. The broker exposed omitted false-valued
`mem_storage` and `headers_only` fields; the validator now compares effective
boolean behavior. The namespace was deleted and verified absent. The local `nats-py`
and Ruff tools were installed in temporary virtual environments, not system Python.
The deploy main commit `1076584` passed Ruff formatting/lint and both gateway and
service image build/publish checks. The platform main commit `b047a00` passed Ruff,
unit tests, container build, and publish. See the [deploy checks](https://github.com/myota-platform/myota-deploy/commit/107658467eb708981322649e448832e272daccbb/checks)
and [platform checks](https://github.com/myota-platform/myota-platform/commit/b047a00e15ecc619e3589fffee37a1aa779ff59a/checks). The live topology and producer/consumer runtime paths were not modified.

The isolated test ran in a temporary namespace on the host's K3s cluster. It
used an emptyDir-backed NATS pod and the already-deployed application image with
the working-tree provisioner source mounted read-only. It created no domain
events or application data. The temporary namespace and its resources were
deleted and verified absent after the test. Docker is not installed in the
executor, so Compose-based tests were unavailable. Test-only finite values were
8 MiB/10,000 messages/1 MiB per event for facts and 4 MiB/1,000 messages/1 MiB
per command for each work stream; Activity and Geodata test max ages were seven
and 30 days. These are harness inputs only, not selected production limits.

## 10 October 2026 — producer source audit

`myota-contracts/contracts/event-registry.json` now records repository-relative
source paths for all 68 registered fact types. The new
`myota-contracts/scripts/audit_workspace_event_sources.py` verified each path
against the literal event type in its owner source file, then scanned Python source
in the five authoritative Identity, Programme, Activity, Geodata, and Operations
repositories for event-like literals without a disposition. Result: 68 facts verified, all six
legacy Geodata work source types mapped in the work registry, and no
undispositioned Python event-like source literals. The two recovery event types are sourced from
`myota-geodata-service/migrations/015_jetstream_worker_dispatch.sql` and its
synchronized deploy migration. This audit establishes source coverage, not complete
payload schemas or runtime enforcement.

The contracts-owned CI workflow now checks out all five producer repositories and
runs this audit. [Contract CI run 38033202206](https://github.com/myota-platform/myota-contracts/actions/runs/38033202206)
passed; its log reports 68 verified facts, six legacy source types classified, and no undispositioned Python event-like literals. The platform mirror commit
`038cd90` passed tests, Ruff, and its container check. Locally, two registry tests, Ruff, workflow
YAML parsing, and mirror equality checks passed. The registry
was synchronized with `scripts/sync_contract_mirrors.py`; no producer, consumer, or
live stream changed.

The exact Phase 1 prompt used remains the copyable
[Phase 1 prompt in the migration plan](../nats-event-migration-plan.md#chatgpt-prompt--phase-1).

## 10 October 2026 — Identity payload contract source pass

The next contract step adds payload schemas for all 19 registered Identity
facts, derived from the event callsites in
`myota-identity-service/identity.py` and the common envelope construction in
`myota-identity-service/common.py`. The registry marks each schema
`source-derived-identity-callsite` and records a data classification. Schemas
allow additive properties so current payloads can be documented without
rejecting compatible extensions. Account/callsign details, login email and
remote address, and authorization events are classified as personal or
security-sensitive. The service-token-issued schema includes service and scopes
only; it excludes the returned access token.

Contracts CI now runs the registry/schema unit tests. Local verification passed:
two unit tests, Ruff lint and format check, generation of all 68 event schemas,
JSON parsing and required payload-shape assertions for all 19 Identity schemas,
the workspace audit (68 facts verified, six legacy work types mapped, zero
unclassified Python literals), and byte-for-byte registry/schema/doc mirror
comparison with `myota-platform`. Contracts CI run
[38033723013](https://github.com/myota-platform/myota-contracts/actions/runs/38033723013)
passed, including the new schema test step and Ruff. Platform mirror commit
`529631c` passed its unit tests, Ruff, and container workflow
([run 38033742782](https://github.com/myota-platform/myota-platform/actions/runs/38033742782)).
No producer, consumer, broker, or deployed
service behavior changed. Our Identity owner/privacy review of data minimization
and retention remains open. At the time of this first payload pass, Programme,
Activity, Geodata, and Operations payload schemas were still open; the following
section records the Programme source pass. The existing Phase 1 prompt remains
the sole phase prompt; this is a contract-only continuation.

## 10 October 2026 — Programme payload contract source pass

The Programme contract step adds payload schemas for all 12 registered facts,
derived from event callsites in `myota-programme-service/programmes.py`.
Schemas cover programme create/update/archive, entity-category catalogue and
assignment changes, content review/publication, and policy-draft save and
publication. Content and policy schemas include their current review-history
fields and classify reviewer/publisher metadata as internal. The producer does
not validate the structure of programme `rules`, `theme`, `oidc`, content
`value`, or policy `schema`; these fields remain open in the schemas rather than
being assigned unsupported constraints. Nested legacy catalogue entries allow
their observed older fields. All payload objects allow additive properties.

The registry marks these entries `source-derived-programme-callsite`; generated
schemas include the evidence and data classification extensions. Contracts tests
now assert all 19 Identity and 12 Programme facts have source-derived payload
shapes. Local checks passed: two registry/schema tests, Ruff lint/format,
regeneration of all 68 event schemas, the workspace source audit (68 facts,
six legacy work types, zero undispositioned Python event-like literals), and
byte-for-byte sync of registry, schemas, and event docs to the platform mirror.
No runtime, consumer, or live broker behavior changed. Joint review of Identity
and Programme payload fields remains open, along with Activity, Geodata, and
Operations schemas. The same Phase 1 prompt continues to apply.
