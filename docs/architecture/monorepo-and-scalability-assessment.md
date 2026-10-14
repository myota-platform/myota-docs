# Monorepo and future-scale assessment

**Status:** Proposed architecture decision, not implemented

**Assessment date:** 2026-10-14
**Scope:** Repository structure, application boundaries, scalability, and language
selection. No runtime behavior, API, schema, or deployment was changed for this
assessment.

## Decision summary

Adopt a **single application-source monorepo over time**, while keeping the
existing service processes, APIs, databases, frontend applications, and
independent deployment/scaling boundaries. A monorepo is a source-management
choice; it does not imply a single application process or shared domain
database.

Keep the current API-first product and both frontends. Keep asynchronous
cross-service work on NATS JetStream where a durable event or work queue fits.
Keep synchronous HTTP for requests that need an immediate answer, such as
authentication, validation, reads, and accepting a command and returning its
job identifier. Do not replace a durable job queue with request/response merely
to reduce repository count.

Do not rewrite the backend from Python based on a general impression that it is
slow. Current source shows bounded threaded HTTP servers, separately deployed
workers, PostgreSQL pools, resumable object-store uploads, and batch QSO writes.
These are useful scaling foundations. The evidence reviewed here contains no
representative comparative benchmark that establishes Python as the limiting
factor. Keep Python for the first consolidation; consider Go for an isolated
worker only if representative profiling identifies a CPU or memory bottleneck.
Continue using TypeScript/JavaScript for the existing browser applications.

Treat **hundreds of concurrent users and thousands of activity uploads per
day** as a future planning envelope, not a measured capacity claim. Daily
average upload count alone does not define peak load: file size, burstiness,
processing cost, query mix, and the completion-time objective determine the
required capacity.

## Current architecture and evidence

The current organization separates domain ownership and deployment artifacts
but also maintains synchronized copies in integration/deployment repositories.
Those copies are a larger simplification opportunity than combining the
services into one runtime.

| Current owner | Evidence in the source repositories | Target boundary to preserve |
|---|---|---|
| Identity | `myota-identity-service/identity.py`, `run_identity.py`, `migrations/`, `tests/`; `run_identity.py` uses `ThreadingHTTPServer` | Identity API, Identity-owned schema/migrations in Core PostgreSQL, independently scalable process |
| Programme | `myota-programme-service/programmes.py`, `run_programmes.py`, `migrations/`, `tests/` | Programme API and Programme-owned schema/migrations in Core PostgreSQL, independently scalable process |
| Activity | `myota-activity-service/activity.py`, `activity_repository.py`, `activity_worker.py`, `adif_retention_worker.py`, `migrations/`, `tests/` | Activity API, upload/job acceptance, Activity-owned PostgreSQL, separately scalable workers |
| Geodata | `myota-geodata-service/geodata.py`, `geodata_import_worker.py`, `geodata_pipeline.py`, `import_formats.py`, `migrations/`, `tests/` | Geodata API, PostGIS ownership, object-backed intake, separately scalable queue workers |
| Gateway and integration runtime | `myota-deploy/services/dev_server.py`, `services/common.py`, `services/runner.py`; the corresponding `myota-platform/services/` files are integration copies | Gateway remains an API boundary; shared HTTP/runtime helpers have one source |
| Contracts | `myota-contracts/contracts/openapi.yaml`, `contracts/event-registry.json`, `contracts/schemas/`, `contracts/python/`, `contracts/route-inventory.json` | Versioned API/event contracts and generated clients remain explicit compatibility boundaries |
| Deployment | `myota-deploy/deploy/helm/myota/`, `compose.yaml`, `db/migrations/`, `.github/workflows/`; `myota-platform/` has corresponding runnable integration and mirror files | Helm, Compose, migrations orchestration, and integration checks have a single canonical source |
| Admin UI | `myota-admin-web/src/`, `package.json`, `Dockerfile`, frontend tests | Keep authenticated Vue/TypeScript administration as a separately built/deployed frontend |
| Participant UI | `myota-web/web/`, `Dockerfile`, frontend workflow | Keep the public participant frontend and its API behavior |
| Email | `myota-email-service/README.md` only | Decide the service contract and delivery semantics before implementation; include its first implementation in the application monorepo |
| Documentation and organization governance | `myota-docs/docs/`, `.github/profile/`, `.github/workflows/` | Keep central architecture documentation and organization-wide policy separately governable |

Additional evidence for the current scaling shape:

- `myota-activity-service/README.md` documents ADIF intake, asynchronous
  parsing, PostgreSQL `COPY`, Activity job ownership, bounded API/database
  pools, and independently deployed workers.
- `myota-geodata-service/README.md` documents resumable, user-bound S3
  multipart uploads; durable outbox handoff; streaming import decoders;
  batch checkpoints; lease recovery; and independent API and worker
  deployments. It also explicitly records remaining per-format peak-RSS and
  kill/redelivery qualification work.
- `myota-deploy/deploy/helm/myota/values.yaml` configures separate Activity API
  and worker replica counts, Geodata work replicas, database-pool bounds,
  reserved database connections, and separate persistent database, NATS, and
  object-store components. Existing settings are deployment choices, not proof
  of future capacity.
- `myota-deploy/compose.yaml` builds many runtime components from the same
  integration context, while some Geodata/Activity images build from service
  repositories. The deployment source has copies under `services/`; the
  runnable `myota-platform` repository has a further integration/mirror tree.
- `myota-docs/docs/architecture/repository-map.md` identifies each domain
  owner and calls `myota-platform` an integration bootstrap with synchronized
  mirrors rather than a domain owner.
- The NATS decision remains governed by
  `myota-docs/docs/operations/messaging/nats-event-migration-plan.md` and
  `myota-docs/docs/operations/messaging/nats-event-migration-inventory.md`.
  JetStream durables distribute work and support bounded replay; PostgreSQL
  owner rows remain authoritative. NATS is not the business database or a
  long-term event archive.

The implementation uses three physical database boundaries: Core PostgreSQL
contains Identity- and Programme-owned schemas, Activity PostgreSQL contains
Activity data, and Geodata PostGIS contains Geodata data. Preserve schema
ownership in either repository structure. Do not introduce cross-service
database writes or foreign keys as a side effect of monorepo consolidation.

### Other simplification candidates

- `myota-identity-service/myota_logging.py`,
  `myota-programme-service/myota_logging.py`,
  `myota-activity-service/myota_logging.py`, and
  `myota-geodata-service/myota_logging.py` currently contain the same logging
  helper. Consolidate it into a small versioned internal package after the
  monorepo lands; retain per-service configuration and tests.
- Identity and Programme `common.py` are the same implementation, while
  Activity and Geodata have different variants. The
  `myota-deploy/services/common.py` and `myota-platform/services/common.py`
  copies are also the same. Do not merge these files by name: extract only
  demonstrably shared transport primitives, keep domain-specific persistence,
  request policy, and upload handling with their owners.
- Activity currently combines activity and award APIs on one process/port:
  `myota-activity-service/activity.py` composes `AwardsHandler`, and its README
  explicitly documents the shared port. Keep this deployment simplification
  unless separate scaling, release, or fault-isolation evidence justifies a
  split.
- The three database migration sets are owner-specific, but their deployment
  copies are listed in `myota-contracts/contracts/runtime-mirror-map.json`.
  Keep one source migration per owning database and package migration
  execution separately; do not combine all schemas into a single database
  migration chain.
- The old Operations observability service is already archived. Do not
  recreate its polling/history API as part of a monorepo migration; use the
  current OpenTelemetry/Grafana ownership documented in
  `myota-docs/docs/observability/`.
- Keep the public and admin frontends as separate artifacts. They have
  different audience, authentication, route surfaces, and release concerns;
  sharing the Git repository does not require merging their UI code.

## Monorepo decision

### Recommended shape

Create one `myota` application repository with stable, separately built
components and a path-aware build graph:

```text
apps/
  gateway/
  identity/
  programmes/
  activity/
  geodata/
  email/                 # add when implementation begins
web/
  admin/
  participant/
contracts/
  openapi/
  events/
  generated-clients/
deploy/
  helm/
  compose/
  migrations/            # one canonical copy per owning database
integration/
  smoke/
  contract-checks/
  mirror-audit-retirement/
packages/
  python-common/          # only stable cross-service primitives
```

Keep `myota-docs` as the canonical documentation repository and `.github` for
organization-wide instructions/profile material. Keep each service as a
distinct process, image, health/readiness surface, resource profile, and
rollout unit. Build only components affected by a change, plus their actual
dependents and required integration gates. Retain per-service tags or immutable
digests so a release can promote one component without forcing a platform-wide
rollout.

The monorepo should contain one source copy of each implementation, contract,
migration, and deployment definition. Remove `myota-deploy/services/` and
`myota-platform/` runtime mirrors only after all builds and integration tests
consume the canonical source. Move the unique runnable vertical-slice and
cross-service smoke coverage from `myota-platform` into `integration/` before
retiring that repository. Do not delete the bootstrap's useful behavior merely
because its source code is duplicated.

Move the future email service into `apps/email/` before it accumulates a
separate deployment and release process. Keep its runtime, queue consumer,
SMTP configuration, and observability isolated as a separately deployed
component. Its email delivery guarantees and idempotency must be designed
explicitly; repository consolidation does not make SMTP delivery exactly once.

### Why this is preferable to the current split

- A change spanning an API schema, producer, consumer, generated clients, and
  Helm values can be reviewed and checked in one atomic commit.
- Path-aware CI can build/test changed components without building every
  image, while a cross-service contract/integration job checks compatibility.
- Contract generation and mirror drift checks become ordinary same-revision
  checks rather than hand-synchronized repository operations.
- One canonical copy removes the current risk that `myota-deploy/services/`
  and `myota-platform/` diverge from service-owned source.
- Domain and data boundaries remain visible as top-level applications and
  independently owned database migrations. A single Git repository does not
  merge database schemas or runtime failure domains.

### Alternatives considered

| Option | Assessment |
|---|---|
| Keep all repositories and continue mirroring | Lowest immediate migration cost, but preserves multi-repository coordination and duplicate-source drift. Appropriate only if repository-level access/security or release ownership requires independent repositories. |
| One repository and one monolithic service | Reject. It couples scaling, fault isolation, releases, resource limits, and database access without a demonstrated need. |
| One application monorepo with independent services and frontends | Recommend. It improves cross-component change coordination and removes mirrors while retaining the current API, data, and deployment boundaries. |
| Merge databases or let services share domain tables | Reject. It breaks current data ownership, migration isolation, and the ability to scale or recover each owner independently. |

The principal cost is larger repository-wide CI and broader contributor access.
Mitigate it with path-filtered jobs, explicit code ownership for sensitive
areas, required contract/integration gates for cross-boundary changes, and
separately pinned artifact digests. Avoid building a custom monorepo framework
until simple workspace tooling proves inadequate.

## Future-scale design

### API and frontend

- Keep gateway and domain APIs horizontally scalable and stateless. Persist
  sessions, idempotency, and workflow state in their owning stores rather than
  process memory. A browser should continue to use the existing API contracts;
  no frontend rewrite is required by this assessment.
- Bound request concurrency, body sizes, timeouts, and per-request resource
  use. Use admission control and clear `429`/retry guidance when bounded worker
  or storage capacity is exhausted. Avoid allowing a large import or report to
  consume all API threads.
- Scale each API independently using observed request rate, latency, CPU,
  memory, and saturation. A target of hundreds of connected users does not by
  itself imply hundreds of simultaneous database connections or a particular
  replica count.

### Upload and asynchronous processing

- Preserve the resumable multipart object-store upload path. Stream bytes and
  checksums in bounded chunks; do not buffer full uploads in API memory or
  write them to a shared pod filesystem. Keep object size, upload duration,
  active sessions per user, and abandoned multipart cleanup bounded.
- Return an accepted job/resource identifier after the upload is safely stored
  and the owner-database transaction/outbox row is committed. Expose status,
  progress, and terminal errors through the existing API resource model.
- Keep JetStream as transport and bounded work distribution. Bound pull batch
  sizes, in-flight ACKs, worker concurrency, message size, retry count, and
  maximum age/bytes. Use separate consumer groups for independent work and
  retain stable job IDs, owner-row leases, and idempotent database effects.
  Queue age and oldest-job latency are better worker autoscaling inputs than
  CPU alone. Do not use a shared broadcast event stream as a command queue or
  as the canonical history store.
- Keep the three outbox relays isolated by database. Scale relay concurrency
  only within each database connection budget, using short transactions,
  bounded batches, and safe row claiming. PostgreSQL outbox ownership already
  provides the transactional handoff; adding Redis solely to coordinate relay
  claims would add another failure and consistency surface without replacing
  that transaction.
- Keep worker classes independent: ADIF/QSO ingestion, award/report/PDF work,
  Geodata import decoding, and retention/maintenance have different CPU,
  memory, I/O, and latency profiles. Give expensive parsing a bounded worker
  pool so it cannot starve notifications or interactive APIs.

### Data stores and capacity controls

- Preserve database-per-owner boundaries. Budget the total connections across
  API replicas, worker replicas, relays, scheduled jobs, migration jobs, and
  operator reserve. Enforce this as a deployment invariant; raising PostgreSQL
  `max_connections` is not a substitute for a connection budget because
  PostgreSQL allocates resources based in part on that setting.
- Monitor pool wait time, active/idle connections, transaction duration,
  lock wait, slow queries, table/index growth, WAL/checkpoint pressure, and
  storage latency for each database. Add a pooler only if measured connection
  churn or fan-out requires it and transaction semantics are validated.
- For Activity, keep bulk QSO writes batched and idempotent. Measure indexes,
  query plans, table growth, and award/statistics recalculation cost at the
  projected retention horizon. Partition large tables only after measured
  query/maintenance benefits justify the added operational complexity.
- Keep PostGIS geometry/index behavior behind Geodata APIs. Qualify expensive
  spatial searches and import batch sizes with representative data and query
  plans; do not let broad imports monopolize interactive review queries.
- Adding API replicas does not make PostgreSQL, PostGIS, JetStream, or
  SeaweedFS highly available. Plan stateful capacity, replication, backups,
  restore tests, and failure-domain placement separately. JetStream retention
  remains bounded for delivery/replay needs; PostgreSQL and object storage
  remain the source of truth for their owned data.

### Measurements required before capacity commitments

Before setting production replica limits or promising a completion SLO, define
and record:

1. Peak concurrent authenticated sessions and request-rate/route distribution.
2. Upload size distribution, peak simultaneous uploads, upload burst window,
   and accepted file formats.
3. Processing-time objective and p95/p99 for upload acceptance, import
   completion, and QSO ingestion; define whether these are interactive or
   background objectives.
4. QSO rows per upload, parsing CPU/RSS, PostGIS feature/vertex distribution,
   and object-store throughput.
5. Retention periods, expected data growth, report/query patterns, and restore
   objectives.
6. Per-service database connection budgets and total stateful storage/IOPS
   limits.

Qualify a burst that is materially above expected steady state, concurrent
uploads mixed with interactive API traffic, worker restarts/redelivery, broker
or database slowdown, and backlog recovery. Record throughput, p95/p99
latency, error rate, queue age, memory high-water mark, pool wait, and recovery
time. The current small-cluster observations are not capacity evidence for
this future envelope.

## Language assessment

| Option | Strengths for MyOTA | Costs and limits | Recommendation |
|---|---|---|---|
| Python (current services) | Existing domain logic and tests; mature PostgreSQL, NATS, geospatial, and data-format libraries; fast delivery. The API uses threaded HTTP servers and workers are separate processes. | CPU-heavy Python code can consume a worker's CPU and delay other work; runtime and memory must be profiled. Package/native-library compatibility remains part of deployability. | Keep as the default during repository consolidation. Profile API and worker workloads before changing language. |
| Node.js / TypeScript | Fits the existing Vue/TypeScript Admin UI and browser stack; strong async I/O model and a natural option for a lightweight gateway/BFF if that service becomes a measured bottleneck. | CPU-heavy parsing or synchronous callbacks block the event loop; moving backends introduces another runtime and duplicates client/domain logic. Existing Python/PostGIS integrations would need replacement or a network boundary. | Retain for frontends. Do not rewrite backend services wholesale. Consider only an IO-bound edge component if profiling and ownership justify it. |
| Go | Strong fit for bounded concurrent network workers and standalone services; static binaries and built-in CPU/heap/trace profiling can simplify deployment and diagnosis. | A rewrite still has to preserve retries, transactions, API compatibility, observability, and domain behavior. Geospatial/file-format library coverage and native bindings must be proven for the required inputs. | Candidate for a narrow worker pilot only after Python profiling demonstrates a bottleneck and a representative prototype passes parity and resource gates. |

The Node.js documentation explains that CPU-bound callbacks can block the
event loop; its nonblocking I/O model is not a general CPU-speed guarantee.
Go's own diagnostics guidance recommends profiling, and its GC guide calls
out allocation rate as a major tuning factor. Python now offers optional
free-threaded builds, but the current production base images use standard
Python 3.12, and extension compatibility would have to be checked before
relying on that mode. These language facts support measurement, not a claim
that one runtime is already faster for this workload.

### Language qualification gate

If profiling identifies a candidate hotspot, compare an isolated Go or
TypeScript implementation against the current Python path using the same
contract, input corpus, database/broker semantics, and idempotency behavior.
Measure throughput, p95/p99 end-to-end latency, CPU-seconds per unit of work,
peak RSS, allocation/GC behavior, database query count, failure recovery, and
operator effort. Preserve the existing implementation as the reference until
all compatibility and recovery checks pass. Port one worker or bounded module
at a time; do not combine language migration with monorepo cutover.

## Proposed phased path

This is a proposal. It does not authorize runtime or API changes. Each phase
needs a separately scoped implementation prompt and verified exit evidence.

### Phase A — Monorepo boundary and migration design

- Confirm all canonical owners and unique behavior in `myota-platform`,
  `myota-deploy`, and `myota-contracts` against current source.
- Define monorepo access, code ownership, release/tagging, generated-client,
  artifact naming, and protected-branch rules.
- Define API, event, database, and deployment parity checks. Capture current
  API route inventory and database migration checksums as migration baselines.
- Exit when every tracked file is classified as canonical source, generated
  output, deploy configuration, test/fixture, mirror to retire, or historical
  evidence; no unique integration check is unassigned.

### Phase B — Import history and establish tooling

- Import active service, contract, UI, deploy, and email repository histories
  under the agreed layout; preserve authorship and tags where practical.
- Keep docs and organization governance external. Add path-aware CI, dependency
  graph, ownership rules, formatting/type/test/build jobs, contract generation,
  and a mirror audit. CI must not silently skip a required cross-boundary gate.
- Exit when all components build from canonical paths and the existing
  repositories still build the same immutable artifacts from their old source.

### Phase C — Remove duplicate sources, not service boundaries

- Switch Deploy and integration checks to the monorepo canonical service,
  contract, migration, and chart sources.
- Compare mirror checksums and generated OpenAPI/event artifacts. Retire a
  mirror only after no workflow, image, local setup, or operator runbook reads
  it.
- Move unique integration bootstrap tests into `integration/`; retire the
  `myota-platform` source only after its behavior is covered and migration
  evidence is retained.
- Exit when no runtime/migration/contract mirror is edited or required, all
  owners still have isolated schemas and processes, and a clean checkout can
  reproduce CI artifacts.

### Phase D — Release cutover with parity

- Release each service independently from monorepo paths with the existing
  immutable image digest and promotion controls. Preserve APIs, event schema
  compatibility, queue names/filters, migrations, secrets, and frontend paths.
- Cut over one component at a time through the existing test and release
  gates. Keep the previous artifact available for rollback; do not require an
  all-at-once fleet rollout.
- Exit when route/contract checks, data ownership checks, worker recovery,
  object upload, frontend smoke, observability, rollback, and clean artifact
  provenance all match or exceed the existing behavior.

### Phase E — Profile and optimize the future-scale envelope

- Establish representative data/load profiles and stateful capacity budgets.
- Run non-production burst, mixed-upload, backlog-recovery, and fault tests.
- Optimize the measured bottleneck. Evaluate Go only for a proven hot path;
  keep the public/admin UI behavior and APIs unchanged.
- Exit when agreed p95/p99 objectives and recovery criteria pass with measured
  headroom and without exhausting the database, NATS, or object-store budgets.

## Risks and guardrails

- **A larger blast radius from shared repository CI:** require per-component
  ownership, path-sensitive jobs, branch protection, and provenance for each
  image. A deployable component must remain independently revertible.
- **Accidental coupling through shared packages:** allow only small technical
  primitives and generated clients. Domain rules, tables, and repositories
  remain private to their owning service.
- **Monorepo migration hides behavior changes:** do not combine path moves
  with API/schema redesign or a language rewrite. Keep separate parity checks
  and phased releases.
- **Autoscaling amplifies database pressure:** calculate total maximum pool
  connections across all replicas before raising HPA limits. Scale workers
  against queue age while honoring per-database admission budgets.
- **NATS is mistaken for the archive:** use bounded stream retention and
  replay policy only for the documented recovery window. Business state and
  source objects remain in the owner database/object store.
- **Premature table partitioning or service decomposition:** first measure
  query plans and worker/API saturation at representative data volumes.

## Open evidence and decision owners

No measured capacity or comparative language benchmark was found in the
reviewed current source. This assessment therefore recommends a structure and
qualification method, not a numeric replica count, database setting, or
runtime performance claim. Before implementation begins, verify the following
against the live default branches because the repository snapshots can change:

- exact unique integration checks and image build contexts in `myota-platform`
  and `myota-deploy`;
- current canonical owner and checksums for every migration and runtime mirror;
- whether the `myota-email-service` repository has acquired implementation
  beyond its current README-only baseline;
- the future peak/burst profile, workload SLOs, data retention, and restore
  objectives;
- Geodata format-specific memory and kill/recovery qualifications recorded
  in `myota-geodata-service/README.md` and
  `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`.

**Recommended decision:** approve the application monorepo direction with
independent deployables and unchanged frontend/API behavior; retain Python
until profiling justifies a narrowly scoped migration; use Go as the first
backend alternative to evaluate for a proven CPU/memory-bound worker; do not
select Node.js for CPU-heavy ingestion solely because it is JavaScript.

## References

- [Repository ownership map](repository-map.md)
- [Current system state](current-state.md)
- [Platform architecture](overview.md)
- [Geodata horizontal-scaling roadmap](../geodata-horizontal-scaling-roadmap.md)
- [NATS event migration plan](../operations/messaging/nats-event-migration-plan.md)
- [Node.js: Don't block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Go: Diagnostics and profiling](https://go.dev/doc/diagnostics)
- [Go: A guide to the Go garbage collector](https://go.dev/doc/gc-guide)
- [Python: Free-threaded Python](https://docs.python.org/3/howto/free-threading-python.html)
- [Kubernetes: Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/)
- [NATS: JetStream concepts](https://docs.nats.io/concepts/jetstream)
- [PostgreSQL: Connection settings](https://www.postgresql.org/docs/current/runtime-config-connection.html)
