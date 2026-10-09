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
Phase 3 still has open bounded-memory and worker termination/failure-injection
gates. The roadmap explicitly says these gates do not authorize increasing
production replicas. The measured Phase 0 baseline is useful but does not
establish broader capacity.

**Why first:** horizontal expansion can multiply a failure mode or create
duplicate/lost processing if upload handoff, worker checkpoints, and recovery
are not proven. This is the clearest explicitly stated rollout gate in the
current backlog.

**Next:** complete Phase 3 bounded processing, worker termination and
concurrent-worker evidence; then review Phase 4 infrastructure and Phase 5
staged-rollout gates. Re-run Phase 2's digest-pinned recovery job when the
deployed SeaweedFS image changes. Coordinate with the NATS and observability
work below.

See [Geodata horizontal-scaling roadmap](../geodata/horizontal-scaling-roadmap.md),
[Phase 0 production evidence](../geodata/evidence/phase0-production-evidence-2026-10-08.md),
and the active [Work in progress index](../work-in-progress/README.md).

### 2. Complete NATS event and work-queue inventory, then close delivery gaps

**Current evidence:** the NATS plan treats the event/work inventory, approved
stream/retention topology, relay safety, consumer idempotency, and Activity
database-polled jobs as phased work. Recent award evidence also records that
the platform integration test was not run against an isolated NATS broker.

**Why second:** accepted asynchronous work must survive relay/worker restarts
without losing or duplicating domain effects. The plan spans all service
outboxes and consumers, so its Phase 0 ownership and topology decisions are
prerequisites to safely broadening or migrating individual flows.

**Next:** reconcile every emitted event and accepted async job, approve the
event-versus-work topology and replay/retention policy, then execute the
relay, consumer, Activity-job, and rollout gates in dependency order. Use the
isolated-broker test gap as a concrete verification item, not as evidence that
production delivery is failing.

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
