# NATS migration Phase 1 contract and topology evidence

**Historical snapshot (9 October 2026):** The status and remaining gates below
reflect the preparation pass at that time. They are superseded by the 10 October
[joint review](phase1-joint-review-2026-10-10.md) and
[Phase 1 completion evidence](phase1-completion-2026-10-10.md).

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

- Owning services must review the source-derived payload schemas for privacy,
  minimization, compatibility, event dispositions, and trusted
  correlation/causation propagation. The Phase 0 source audit found no Operations
  event producer, so an Operations payload schema is not applicable unless it
  begins publishing facts.
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
Contracts CI run
[38034857206](https://github.com/myota-platform/myota-contracts/actions/runs/38034857206)
passed, including the schema step and Ruff. Platform mirror commit `b8c8507`
passed unit tests, Ruff, and its container job
([run 38034874824](https://github.com/myota-platform/myota-platform/actions/runs/38034874824)).
No runtime, consumer, or live broker behavior changed. Joint review of Identity
and Programme payload fields remains open, along with Activity, Geodata, and
Operations schemas. The same Phase 1 prompt continues to apply.


## 10 October 2026 — Activity payload contract source pass

The contracts registry now includes additive source-derived payload schemas for
all ten Activity facts: activation created/closed, QSO recorded, ADIF queued,
entity cascade deleted, and the five award definition/request/issuance/rendering
facts. Evidence was traced to `myota-activity-service/activity.py`,
`myota-activity-service/activity_repository.py`, and
`myota-activity-service/awards.py`. Activity and award payloads contain personal
activity and certificate information; imports expose object metadata. Flexible
programme rules, award conditions, template elements, and nested asset records
remain open in the schema where the producer permits caller-defined structures.
No runtime producer/consumer code or deployed topology changed. Joint owner and
privacy review remains required.

Local verification passed: registry/schema tests 2/2; schema generation; source
audit for 68 facts and six legacy work types with no undispositioned literals;
Ruff lint/format; and byte-for-byte registry/schema/event-doc mirror checks.
Contracts commit `2b8bddaf` and platform mirror commit `bf6f2727` are on `main`. The
GitHub connector returned no combined status checks for these commits, so Actions
results remain unverified. Runtime behavior and deployed topology did not change.


## 10 October 2026 — Geodata payload contract source pass

The contracts-owned registry now includes source-derived additive payload schemas
for all 27 Geodata facts, based on the event call sites in
`myota-geodata-service/geodata.py` and the transactional persistence paths in
`myota-geodata-service/common.py` and `myota-geodata-service/relational_state.py`.
The schemas classify geometry/location and reviewer data, imported source data,
and internal operational metadata. Full entity/review and deletion-job payloads
retain open additive fields because these objects are persisted as source-owned
records and some nested shapes are dynamic.

The source review found a material minimization issue: `geodata.import.preprocessed.v1`
passes the full preprocessing result to `store.event`. That result includes
internal `_records` and `_status` members, and `_records` can contain imported
source features. The source-derived schema currently permits these fields through
its additive properties. Geodata owner/privacy review must decide the safe event
projection and remove internal/large source records before schema enforcement.
This finding does not change runtime behavior in this phase.

Local verification passed: contracts registry/schema tests 2/2; generation of all
68 event schemas; source audit (68 facts, six legacy work types mapped, no
undispositioned Python event-like literals); and exact platform mirror checks for
the registry, event documentation, and generated schemas. Ruff lint/format checks
passed for the changed Python files. Contracts CI passed for Geodata commit
`e34de821` in [run 38042564323](https://github.com/myota-platform/myota-contracts/actions/runs/38042564323);
Activity contract CI passed for `2b8bddaf` in
[run 38040217352](https://github.com/myota-platform/myota-contracts/actions/runs/38040217352).
Platform mirror CI passed for `63cec55e` in
[run 38042622987](https://github.com/myota-platform/myota-platform/actions/runs/38042622987).
Joint Geodata owner/privacy review remains open. No producer/consumer runtime path,
stream, or deploy input changed. A read-only 10 October cluster check reported the
default K3s context, the `spainip-k3s` node Ready, Fleet `myota-deploy` Ready at
commit `042a45b01ec8b94ed4f64bbc1b854744b9f96ed1`, and Helm release `myota`
deployed at revision 157. The deploy runbook sends chart changes through Fleet;
these contracts/docs changes do not change the deployed stack, so no rollout was
triggered. The same Phase 1 prompt remains in use. The Activity and Geodata source-schema
checklist items are complete; owner/privacy review and the Phase 1 exit criteria
remain open.


## 10 October 2026 — delegated joint review and live sizing sample

The workspace owner delegated a joint review to Volker Kerkhoff and Codex as
the two-person project team. The review covered all 68 fact schemas, ten
selected work contracts, role credentials, current broker capacity,
restore/replay, and relay-side provisioning. Per-event classifications and
dispositions are in the [Phase 1 joint review](phase1-joint-review-2026-10-10.md).

Decisions: accept the source-derived schemas as inventory contracts only, with
producer enforcement gated on minimal projections, prohibited-field and size
checks, and compatibility fixtures; require per-role NKey credentials, TLS,
and a separate provisioner identity; set provisional caps of 1 GiB facts,
1 GiB Activity work, and 3 GiB Geodata work (5 GiB total against the 8 GiB PVC,
reserving 3 GiB); restore stream snapshots off-node and recreate consumers from
declarative config; and make one deployment-owned create-only provisioner the
only topology writer. Do not run target provisioning against the mixed legacy
stream before inspection and backup.

Read-only queries from each deployed relay pod found no unpublished outbox
rows. Under the 30-day query predicate, source rows actually span only 2–9
October in core (162 rows, largest 387 bytes), 6–8 October in Activity (3,016,
largest 150 bytes), and 6–9 October in Geodata (15,925, largest 373,607 bytes).
Geodata includes load-test traffic. This is less than eight days of evidence and
measures database JSON, not serialized NATS messages or future command traffic;
the proposed limits are not production-qualified. The NATS PVC is 8 GiB; the
single K3s node was Ready and Helm release myota was revision 157. No stream,
message, durable, or application row changed.

Remaining gates: server/client auth and TLS plus role ACL tests; representative
full-window sizing and pressure recovery; off-node backup and isolated
restore/replay; Helm/Fleet provisioner readiness; safe relay mutation removal;
and a compact versioned Geodata preprocessing fact before enforcement. Phase 1
remains in progress.


## 10 October 2026 — isolated JetStream restore and replay qualification

A disposable K3s namespace ran two NATS 2.10 JetStream servers. Using the
temporary NATS CLI v0.2.3, the test created a finite Limits stream, published
three synthetic messages, and created a durable pull consumer. It acknowledged
sequence 1 and left sequences 2–3 pending, then used stream backup and restored
to the second clean broker. Restore returned the stream with all three messages
and the original durable state (acknowledgement floor 1, two unprocessed).
The restored durable fetched and acknowledged the remaining two. A new
DeliverPolicy.ALL durable replayed all three messages; direct retrieval of
sequence 1 also succeeded. This validates basic stream data and durable-state
snapshot/restore plus independent replay in isolation.

The temporary namespace was deleted and verified absent, and the temporary CLI
and backup files were removed. No production NATS or application database was
used. This same-host test does not qualify off-node backup retention, PVC loss,
database-to-stream reconciliation, authenticated client permissions, or
production recovery objectives. Phase 1 restore/replay exit evidence remains
open for those controls.
