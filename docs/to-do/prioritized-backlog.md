# Prioritized backlog

**Reviewed:** 2026-10-09

This page orders the currently listed open work using evidence in the linked
plans. It is a documentation-based recommendation, not a committed delivery
schedule: no due dates, staffing, external deadlines, or production incidents
are recorded here. The linked roadmap remains the source of detailed scope and
checkbox progress. Revisit this ordering when owners provide delivery dates or
new operational evidence.

## Priority guide

- **P0 — unblock safe operation or expansion:** correctness, recovery,
  authorization, or governance gates that should close before the affected
  production capability is expanded or relied upon.
- **P1 — near-term operational or product foundation:** close an identified
  cross-service gap or establish foundations needed by later work.
- **P2 — planned capability or cleanup:** valuable work without a documented
  current production blocker; sequence behind higher-risk gates and its
  prerequisites.

## P0 — close before the affected rollout or capability expansion

### 1. Qualify geodata upload and worker recovery before scaling

**Current evidence:** Phase 2 API/session recovery across API and SeaweedFS
container restarts passed in isolated CI against the exact SeaweedFS image ID
observed in the K3s deployment; see the [Phase 2 evidence record](../geodata/evidence/phase2-upload-recovery-2026-10-09.md).
Phase 3 is now closed against its documented 16 MiB/250,000-vertex/5,000
feature and 100-feature/32-MiB checkpoint bounds. XML and OSM area guards,
all-format/remote-response/snapshot RSS, maximum-vertex geometry, and forced
worker death/replay at parse, checkpoint, enrichment, and promotion passed in
an isolated disposable K3s test namespace; JetStream graceful drain and
redelivery tests also passed. The later Phase 4 boundary test separately
qualified only the charted API range of two to three replicas. Phase 5 has since
demonstrated an accepted large import during same-node API replacement and an
actual CPU-triggered HPA move from two to three, followed by automatic
scale-down. The ten-entity read comparison showed modest p95 improvement;
same-row edit contention exposed PostgreSQL lock waits and the two-replica p95
missed its target. This does not establish broad capacity, multi-node failover,
independent-row edit consistency, or rollback safety. See the [Phase 3 review](../geodata/evidence/phase3-bounded-preprocessing-2026-10-09.md),
[Phase 4 report](../geodata/evidence/phase4-infrastructure-scaling-2026-10-09.md),
and [Phase 5 report](../geodata/evidence/phase5-staged-rollout-2026-10-09.md).
The measured Phase 0 baseline is useful but does not establish broader capacity.

**Why first:** horizontal expansion can multiply a failure mode or create
duplicate/lost processing if upload handoff, worker checkpoints, and recovery
are not proven. This is the clearest explicitly stated rollout gate in the
current backlog.

**Next:** keep the Phase 2/3/4/5 bounded evidence linked to the deployed
components; investigate same-row lock waits and verify independent-row writes;
complete Phase 5's broader capacity, node/storage-failure, canary and rollback
gates before expansion beyond the charted two-to-three API range; and rerun
Phase 2's digest-pinned recovery job when the deployed SeaweedFS image changes.
Coordinate with the NATS and observability work below.

See [Geodata horizontal-scaling roadmap](../geodata/horizontal-scaling-roadmap.md),
[Phase 0 production evidence](../geodata/evidence/phase0-production-evidence-2026-10-08.md),
and the active [Work in progress index](../work-in-progress/README.md).

### 2. Implement NATS contracts and close delivery gates before migration

**Current evidence:** Phase 0's exhaustive inventory and decision record are
complete. [ADR-0008](../architecture/decisions/0008-nats-jetstream-event-and-work-topology.md)
selects `MYOTA_EVENTS` with bounded Limits retention for domain facts, plus
separate `MYOTA_ACTIVITY_WORK` and `MYOTA_GEODATA_WORK` WorkQueue streams. Phase
1 has started: the contracts repo lists 68 facts and ten work commands and has
outer-envelope schemas; a create-only provisioner validates finite limits and
configuration drift. The registry now maps all 68 facts to producer source files;
a workspace audit found no undispositioned Python event-like literals across
five service repositories. The contracts source-audit workflow passed; platform unit tests and
Ruff format/lint CI pass. The expanded consumer configuration also passes four
focused tests and an isolated create/idempotency run against a disposable broker.
The 19 Identity, 12 Programme, 10 Activity, and 27 Geodata event payloads now
have additive, source-derived schemas and data-classification metadata. Activity
and Geodata local schema assertions pass; their GitHub Actions status is not
available from the connector. Joint owner/privacy review remains open. Geodata
preprocessing currently includes internal `_records` and `_status` data in its
event payload; minimization review is required before enforcement. Operations
payload schemas remain open.
Earlier platform image-build attempts timed out at Docker Hub; the latest deploy
and platform main-branch image build/publish checks now pass. The payload schemas are still incomplete. The deployed broker remains a single Interest-retained stream with an 8 GiB PVC,
and no live migration or runtime change has occurred. Read-only sampling found
19,103 outbox rows spanning only
2–9 October and an unusually large Geodata payload; that is not a full-window
capacity forecast. Credentials, numeric limits, broker restore/replay evidence,
and relay-side provisioning behavior remain open. The contracts #2, deploy #4,
platform mirror #1, and organization profile #1 PRs are merged; merge SHAs and CI
outcomes are recorded in the [Phase 1 evidence](../operations/messaging/evidence/phase1-contract-topology-2026-10-09.md).
**Why second:** accepted asynchronous work must survive relay/worker restarts
without losing or duplicating domain effects. The selected topology is now
recorded, so the remaining priority is to establish the contract and safe
provisioning, close the evidence gate for each affected flow, and verify relay,
consumer, and recovery behavior before cutover.

**Next:** close Phase 1 owner/privacy review of the source-derived payloads and
subscriber contracts, minimize the Geodata preprocessing event, establish secure
per-role broker access, derive limits from representative traffic and the
allocated volume, and run restore/replay tests on an isolated broker. Remove
relay-side topology mutation before provisioning can be used. Do not begin
Phase 2 behavior changes until Phase 1 exit criteria pass, and do not change a
producer/consumer path until its evidence gate is met or explicitly accepted
with a time-bounded recovery plan.

See [NATS event migration plan](../operations/messaging/nats-event-migration-plan.md),
[JetStream operations status](../operations/messaging/jetstream-admin-status.md),
and [award designer delivery evidence](../domain/awards/evidence/programme-awards-2026-10-09.md).

### 3. Establish programme governance, jurisdiction, and eligibility controls

**Current evidence:** the programme configuration gap analysis labels
programme identity/ownership, locale/content policy, jurisdiction/approver
scope, structured eligibility, and identity/privacy areas as P0. These are
still unchecked in that roadmap.

**Why third:** programme policy determines who can administer and review work,
which entities and activities qualify, and what public rules participants
rely on. Implementing additional programme self-service without versioned
ownership, jurisdiction, and eligibility policy risks inconsistent decisions
across programmes.

**Next:** start with ownership/lifecycle and audit semantics, then stable
jurisdiction codes and scoped approver decisions, followed by versioned
eligibility rules and publication snapshots. Confirm authorization boundaries
with the owning services before exposing controls in Admin UI. Treat programme
locale/content policy as a separate dependent track; it is distinct from the
admin interface translation roadmap.

See [Programme configuration gap analysis](../domain/programmes/configuration-gap-analysis.md)
and the [administration workflows index](../domain/administration/README.md).

## P1 — close identified operational visibility gaps

### 4. Restore Identity and Programme metrics coverage

**Current evidence:** the observability page records HTTP 404 responses from
the configured Identity and Programme `/metrics` targets and Prometheus
`up=0` for both. It explicitly says this is a metrics coverage gap, not proof
that either service API is down.

**Why here:** missing service/request series weaken diagnosis and alerting
during production incidents. This should be fixed and verified before relying
on fleet-wide service metrics, while keeping the distinction between
observability and application availability clear.

**Next:** align each service's actual metrics endpoint/transport with scrape
configuration, then verify non-404 responses, live series, and dashboard
coverage from the running cluster.

See [Observability live verification and known gap](../observability/overview.md#live-k3s-verification-and-known-gap)
and the [observability evidence](../observability/evidence/2026-10-09.md).

## P2 — sequence after the rollout and operational gates

### 5. Define the typed future award-condition model

**Current evidence:** the roadmap proposes POTA-like activation, worked-entity,
Maidenhead, geographic, milestone, and MyOTA-specific diversity conditions.
Its Phase 0 still needs evidence and semantic decisions; later phases depend
on reliable worked-entity resolution and versioned geodata/reference evidence.

**Why here:** it is a substantial product capability with data-provenance and
recalculation dependencies. Agree on the condition semantics and evidence
contracts early, but sequence broad evaluator/UI implementation behind the
relevant Activity and Geodata prerequisites.

**Next:** complete Phase 0 evidence and definitions, then decide which reference
data and QSO-resolution work must precede evaluator implementation.

See [Future award-condition model and implementation roadmap](../domain/awards/future-condition-model-roadmap.md).

### 6. Complete the admin UI internationalization foundation and translations

**Current evidence:** the roadmap is proposed and all phases are unchecked. It
calls for locale policy and inventory before introducing translation catalogs
or migrating screens.

**Why here:** this improves usability and maintainability but no current
operational or release blocker is documented. Its discovery phase can be
scheduled independently; bulk UI migration should follow an agreed language
and review policy.

**Next:** complete the locale/string inventory and choose the source locale,
supported locales, fallback, persistence, and translation review ownership.

See [Admin UI internationalization roadmap](../domain/administration/i18n-roadmap.md).

### 7. Retire legacy REST aliases after usage-based migration

**Current evidence:** REST consolidation Phases 0–4 are recorded as implemented;
Phase 5 remains planned. The plan requires usage measurement and client
migration before aliases or duplicate contract copies are removed.

**Why here:** current compatibility routes are intentional migration support.
Removing them prematurely could break deployed clients; there is no documented
deadline that makes removal more urgent than reliability and policy work.

**Next:** measure alias usage, identify remaining clients, publish deprecation
and sunset dates, and remove aliases only after migration evidence meets the
plan's criteria.

See [REST API consolidation plan](../domain/api/rest-consolidation-plan.md).

### 8. Address longer-term programme and charter gaps

**Current evidence:** the charter gap analysis describes longer-term product,
governance, interoperability, and production-readiness work. Detailed
programme gaps not classified as P0 in their own roadmap also remain.

**Why here:** these items matter to product maturity, but their scope needs
owner decisions and prioritization against the concrete operational gates
above. Some programme-configuration subsections already labeled P0 are ranked
separately at item 3 and should not be deferred with this broader group.

**Next:** assign owners and split accepted charter findings into deliverable
roadmaps with dependencies, measurable outcomes, and review dates.

See [Charter delivery gaps](../governance/charter-gap-analysis.md) and
[Programme configuration gap analysis](../domain/programmes/configuration-gap-analysis.md).

## Reprioritization notes

- Move an item up if an owner identifies a dated launch, compliance, security,
  or reliability dependency; record the evidence and date in the source plan.
- Keep rollout gates attached to the capability they qualify. For example,
  geodata recovery evidence blocks scale expansion even if other work proceeds.
- The priority here ranks the eight entries in the To do index. Active evidence
  and gates remain discoverable in the [Work in progress index](../work-in-progress/README.md).
