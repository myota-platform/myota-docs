# Phase 2 relay hardening and event coverage evidence

**Status:** Phase 2 source implementation and exit checks complete on 10 October
2026. This is not evidence of production rollout or completion of consumer/work
stream migration.

## Implemented behavior

- `myota-contracts/contracts/event-registry.json` is the canonical registry for
  68 fact types and ten selected work commands. The generated
  `myota-deploy/services/event_registry.json` and
  `myota-platform/services/event_registry.json` provide the relay's compact
  runtime routes and owner-specific producer names.
- `myota-deploy/services/outbox_routing.py` rejects unknown event types,
  producer/owner mismatches, malformed UUIDs, timezone-naive timestamps,
  non-object payloads, and fact subject overrides. Registered facts use
  `myota.events.<eventType>` with dotted tokens. Six legacy Geodata work source
  types keep their allowlisted current subjects until the work cutover.
- `myota-deploy/services/outbox_worker.py` publishes one claimed row at a time,
  sets `Nats-Msg-Id` to the stable event ID, enforces the configured serialized
  message cap, and marks the database row only after JetStream acknowledgement.
  Retry delay is exponential with bounded jitter and a 300-second ceiling;
  retry exhaustion or contract failure is persisted to the owning database.
  A failed database mark leaves the row pending for safe same-ID republish.
- Relay startup validates the existing `MYOTA_EVENTS` mixed Interest stream and
  all five legacy durables read-only. It does not create or modify streams or
  consumers. `myota-geodata-service/geodata_import_worker.py` no longer deletes
  the retired push durable at startup; durable removal requires an explicit
  reviewed drain and disposition.
- The core, Activity, and Geodata relay Deployments bind to their corresponding
  `core-database-url`, `activity-database-url`, and `geo-database-url` secret
  keys. The same mapping is explicit in Compose. Relay concurrency is one item
  per process; each Deployment has one replica.
- Each relay exposes bounded-cardinality counters and gauges for publish,
  retry, dead-letter, pending count/age, unresolved dead letters, database
  health, and NATS health. Compose and Helm scrape all three relay Services;
  Grafana panels and Prometheus alerts cover backlog, sustained retries,
  unresolved dead letters, and missing relay metrics.
- New core, Activity, and Geodata migrations add unresolved-dead-letter state
  and a redrive audit table. `services/outbox_admin.py` lists redacted metadata
  and redrives only retained source rows, recording the supplied actor/reason.
  Use Kubernetes API authorization/audit as the execution identity boundary;
  the CLI actor field is an audit label, not an independently authenticated
  identity.
- Geodata import retention now excludes unresolved dead letters from age-based
  cleanup. Pending outbox rows remain excluded by the existing
  `published_at IS NOT NULL` cleanup condition.

## Coverage and focused verification

- Workspace source audit: **68** registered fact source types, **6** mapped
  legacy Geodata work source types, and no undispositioned event-like literals
  across Identity, Programme, Activity, Geodata, and Operations.
- Contracts registry suite: **5/5 passed**. Deploy relay/topology suite:
  **17/17 passed**. Geodata import-retention suite: **6/6 passed**.
- The synchronized `myota-platform` integration suite passed **47 tests** with
  one skip for an unconfigured dedicated JetStream service. Its old outbox tests
  were replaced with the deploy-owned contract and read-only topology suites
  after the first post-push CI run exposed stale underscore-subject expectations.
- Activity notification duplicate-delivery regression: **2 passed**, with the
  broker-backed overlap/restart case skipped because that test suite was not
  configured with its own JetStream service. The consumer checks
  `consumer_processed_event`; notification creation also uses the unique
  `event:<eventId>` key so a repeated handler call returns the existing notice
  without enqueueing a second notification job.
- Ruff lint and format checks passed for the changed relay, routing, admin CLI,
  and tests. Helm lint and render passed. Compose, Collector configs, alert
  rules, rendered Helm YAML, and the dashboard JSON parsed successfully.
- **Live rollout — message cap:** Helm initially rendered the numeric
  `outbox.maxMessageBytes: 1048576` as `1.048576e+06`, which the relay's integer
  parser rejects. The deploy-owned value is now the quoted string `"1048576"`
  (commit `ca70e32`) and the platform mirror is synchronized (commit `50d3ad3`).
  Helm render and both repository CI checks passed. During the active upgrade,
  setting that same intended value on the three Deployments restored readiness;
  Fleet later applied the chart value.
- **Live rollout — migration image:** The migrations Job initially used a
  cached image pinned to the old platform digest, which lacked the new
  dead-letter columns. Relay metrics logged `resolved_at does not exist` while
  the workers stayed up. The hook now uses `imagePullPolicy: "Always"` (deploy
  commit `ce8065b`, platform mirror `1444d82`), and the chart is versioned as
  `0.2.14` (deploy commit `c7c904c`). The digest-sync workflow recorded current
  image digest `sha256:858519…` in deploy commit `af5ef1d`; the digest values
  mirror is platform commit `b0552b5`.
- **Live verification:** Migration Job 166 completed with the current pinned
  image. Read-only checks found `resolved_at` and its partial unresolved index
  in all three service databases. Each relay Deployment is 1/1 Ready; its
  metrics endpoint reports database and NATS health `1` and pending rows `0`,
  with no recent database errors. Geodata reports six unresolved dead letters;
  these were left intact for operator inspection. Fleet is Ready with 59/59
  resources and Helm revision 167 is deployed on chart `0.2.14` (Helm describes
  it as a rollback to revision 166). Consumer/work-stream cutover and
  mixed-stream retirement remain open.
- All three redrive migration files applied idempotently against disposable
  PostgreSQL in the test namespace. The real relay published a registered fact
  to its dotted subject, used the event UUID as `Nats-Msg-Id`, and recovered
  after an injected publish-ack/database-mark failure. A repeated same-ID
  publish returned the same JetStream sequence and `duplicate=true`.
- The real relay metrics endpoint returned healthy database/NATS gauges and
  backlog metrics. An unknown registered route was visible in the database DLQ;
  the redrive CLI recorded actor/reason and reset the retained source row for
  retry. Test rows were deleted afterward.
- The isolated release `myota-phase2-test` ran in namespace
  `myota-phase2-test` with disposable NATS/PostgreSQL resources. The namespace
  was removed after verification. No production stream, durable, or database
  object was modified during these checks. The later GitOps rollout and its
  value correction are recorded separately above.

## Live legacy topology inspection

A read-only NATS metadata query on 10 October observed:

| Object | Observed configuration |
|---|---|
| `MYOTA_EVENTS` | File storage, Interest retention, one replica, subjects `myota.events.>` and `myota.geodata.>`, zero stored messages at inspection time. |
| `activity-notifications-pull-v1` | Pull, explicit ACK, filter `myota.events.>`, 60-second ACK wait, ten deliveries, 64 pending. |
| Four Geodata work durables | Pull, explicit ACK, exact legacy Geodata work filters, 120-second ACK wait, 100 deliveries, four pending. |

The Activity durable remains broad because its Interest-retained state cannot be
edited in place safely. Provision and drain a registered-filter successor in
the Phase 3 consumer transition. The live check inspected metadata only and did
not fetch messages or mutate broker state.

## Boundaries and remaining gates

- Source-derived payload schemas are not runtime JSON Schema enforcement.
  Prohibited-field/purpose checks and a compatible projection for
  `geodata.import.preprocessed.v1` remain open. The size limit alone does not
  establish data minimization.
- Existing producers and consumers have not been rolled out to target work
  streams. Four Geodata work durables remain on the mixed legacy stream;
  Activity jobs remain PostgreSQL-polled pending their later work migration.
- Activity side-effect and processed-event checkpoint writes are separate
  transactions. The unique notice key prevents duplicate notices, but a future
  consumer-reliability phase should make the side-effect/checkpoint boundary
  atomic where the service schema permits.
- The live stream retains Interest semantics. The Phase 2 change does not
  convert it to Limits retention or activate the target Helm pre-upgrade
  provisioner. Production watermark comparison and controlled cutover remain
  required before changing the deployed topology.
- JetStream remains a bounded delivery/replay window, not an archive. Database
  idempotency protects retries beyond the configured deduplication window.

## Exit criteria

- [x] Source audit and registry reconciliation found no unregistered event-like
      source or subject; registered producer helpers write to their service-owned
      transactional outboxes.
- [x] Isolated relay restart/failure-window checks proved ack-before-mark and
      same-ID retry after the mark failure; JetStream deduplication was verified
      within its configured window.
- [x] Backlog, oldest age, retry, and unresolved DLQ signals are exposed and
      actionable across the three database-bound relay deployments.

These checks complete Phase 2 implementation evidence. They do not authorize a
production stream-retention change or claim that a producer/consumer path has
been cut over.
