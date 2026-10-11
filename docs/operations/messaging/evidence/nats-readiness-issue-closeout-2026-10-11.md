# NATS migration evidence/readiness issue closeout

**Verified:** 11 October 2026, 20:18–20:22 UTC  
**Scope:** The seven NATS migration evidence/readiness entries in the
organization issue index. Six implementation/evidence issues are completed.
Operations #1 is closed as superseded by the approved monitoring design and the
separate Operations retirement issue. This closeout does not claim production
traffic, multi-node recovery, or production capacity qualification.

## Current live topology

A read-only K3s check used an ephemeral `nats-box` pod in namespace `myota`.
It queried streams and consumers only; it published no messages. The pod was
deleted by `kubectl run --rm` and verified absent afterward.

| Stream | Subject | Retention | Storage / replicas | Age / bytes / messages / message | Discard | Current state |
|---|---|---|---|---|---|---|
| `MYOTA_EVENTS` | `myota.events.>` | Limits | File / 1 | 30 days / 1 GiB / 500,000 / 1 MiB | New | 0 messages, 0 bytes, 1 consumer |
| `MYOTA_ACTIVITY_WORK` | `myota.work.activity.>` | WorkQueue | File / 1 | 30 days / 1 GiB / 250,000 / 1 MiB | New | 0 messages, 0 bytes, 6 consumers |
| `MYOTA_GEODATA_WORK` | `myota.work.geodata.>` | WorkQueue | File / 1 | 30 days / 3 GiB / 500,000 / 1 MiB | New | 0 messages, 0 bytes, 4 consumers |

The Activity fact durable is `activity-notifications-v1` with 21 exact
registered filters. The ten work durables are the six Activity kinds and four
Geodata commands. `nats consumer ls` reported zero ack-pending and
unprocessed messages for the listed consumers. The three database relay
Deployments, `myota-core-outbox`, `myota-activity-outbox`, and
`myota-geo-outbox`, were Ready.

At the same check, Helm release `myota` was deployed at revision 257
(`myota-0.2.14`). Fleet's Deploy GitRepo was Ready 1/1 at
`f37a1fa8b5f3f5c0eb639c7125fcc9ab0fe323b8`; the MyOTA bundle was deployed
and monitored. All 23 Deployments had 27/27 desired replicas ready. NATS and
NATS Surveyor pods were Ready. This is a point-in-time single-node check.

## Issue disposition

| Repository / issue | Disposition | Evidence and boundary |
|---|---|---|
| [Contracts #1](https://github.com/myota-platform/myota-contracts/issues/1) | Completed | Registry has 68 fact schemas and 10 work-command schemas. Envelope v1, subject/owner/source declarations, unknown-version behavior, and consumer dispositions are checked by the registry tests. A clean-clone run of `python3 -m unittest tests.test_event_registry` passed 7/7 on Contracts commit `1e0992c`. |
| [Identity #1](https://github.com/myota-platform/myota-identity-service/issues/1) | Completed | The 19 Identity facts have explicit classifications and subscriber dispositions. Activity notifications consumes only the 19 selected Identity facts; there is no Identity-owned subscriber or no-op durable. |
| [Programme #1](https://github.com/myota-platform/myota-programme-service/issues/1) | Completed | All 12 Programme facts have explicit dispositions. No current Programme consumer has an approved business use; no no-op consumer was added. |
| [Activity #1](https://github.com/myota-platform/myota-activity-service/issues/1) | Completed | The notification durable's 21 filters cover 19 Identity and two Geodata facts. Six selected jobs use owner-row job IDs/status and six exact WorkQueue durables. Focused execution/retry/idempotency evidence is in Phase 3/4 records. `NOTIFICATION_SEND` is excluded; its 154 legacy rows were already delivered and synthetic state-only jobs were removed. |
| [Geodata #2](https://github.com/myota-platform/myota-geodata-service/issues/2) | Completed | Phase 5 evidence records transactional state/outbox coupling, stable owner-row identity, the current recovery migration 021, retry/idempotency behavior, and reconciliation as recovery rather than a second dispatcher. Six historical unresolved Geodata DLQ records remain preserved. |
| [Deploy #3](https://github.com/myota-platform/myota-deploy/issues/3) | Completed for accepted scope | Provisioning is drift-checked and the one-time migration flag is disabled. Phase 1/2/6 evidence covers stream ceilings, relay controls, redrive ownership, Compose/Helm, and isolated same-node restore/replay. Helm/Fleet/live state is above. |
| [Operations #1](https://github.com/myota-platform/myota-operations-service/issues/1) | Superseded; closed as not planned | Operations has no NATS connection. NATS Surveyor and Grafana provide broker monitoring. Complete Operations service retirement remains separately tracked by [Operations #3](https://github.com/myota-platform/myota-operations-service/issues/3). |

The Contracts workspace source audit reported: “verified 68 registered fact
types across 5 authoritative repositories; 16 legacy work source types have
registry dispositions; no undispositioned event-like source literals found.”
The Platform copy of `contracts/event-registry.json` was synchronized to the
Contracts-owned registry; Platform's registry test job passed in run
[38171688089](https://github.com/myota-platform/myota-platform/actions/runs/38171688089).

## CI and evidence limits

The latest Contracts freeze workflow,
[run 38171573878](https://github.com/myota-platform/myota-contracts/actions/runs/38171573878),
is not green: it fails at the generic source-mirror audit before the registry
test step, reporting Identity and Programme handler-file drift. The focused
registry suite and workspace event-source audit were run directly and passed.
Platform run 38171688089 passed its test job but failed Ruff formatting in
`services/identity.py`. These are recorded; full cross-repository CI is not
claimed as green. The NATS issue closeout does not mark the separate Identity
and Programme metrics work complete.

The owner explicitly deferred off-node recovery for the current single-node
scope. The disposable same-node PVC restore/replay drill passed; node-loss,
cluster-loss, and off-host recovery were not tested. The owner accepted the
initial finite ceilings while the traffic sample remains shorter than 30 days;
they are conservative ceilings, not a measured 30-day capacity qualification.
The internal-only workload trust boundary remains in effect; reassess NATS
authentication/TLS and role-scoped access before external exposure or
untrusted workloads gain network reachability.

Six unresolved Geodata application DLQ records remain in their owner database.
They were not redriven or purged. Zero current NATS backlog does not establish
historical event completeness; PostgreSQL and owner rows remain authoritative.

## Detailed phase evidence

- [Phase 0 inventory](../nats-event-migration-inventory.md) and
  [Phase 1 contract/topology evidence](phase1-completion-2026-10-10.md)
- [Phase 2 relay evidence](phase2-relay-hardening-2026-10-10.md)
- [Phase 3 consumer evidence](phase3-domain-consumers-2026-10-10.md)
- [Phase 4 Activity work evidence](phase4-activity-work-2026-10-10.md)
- [Phase 5 Geodata work evidence](phase5-geodata-work-2026-10-10.md)
- [Phase 6 stream cutover evidence](phase6-fact-stream-cutover-2026-10-11.md)
- [Current system snapshot](../../../architecture/current-state.md)
- [ADR-0008 topology decision](../../../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md)
