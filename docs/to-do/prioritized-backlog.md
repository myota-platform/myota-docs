# Prioritized backlog

For the latest verified deployment and workstream status, see [Current system state](../architecture/current-state.md). This page lists unfinished work only; completed milestones remain in their source plans and evidence.

**Reviewed:** 2026-10-13 09:15 UTC

This is a recommended order, not a delivery schedule. It records no due dates,
staffing commitments, external deadlines, or production incidents. Roadmaps own
the detailed tasks and checkboxes. Reorder when scope, evidence, or owner
priorities change.

## Priority guide

- **P0 — before the affected capability is expanded or relied upon:** correctness,
  recovery, authorization, or governance gates.
- **P1 — operational visibility or product foundations:** close known service
  gaps and foundations needed by later work.
- **P2 — planned capability or cleanup:** valuable work without a documented
  current production blocker.

## P0 — before expansion or dependent implementation

### 1. Finish baseline readiness before rebuilding the three databases

**Current position:** Core, Activity, and Geodata baselines are merged to `testing`; the unified runner passed fresh, repeat, and readiness-failure checks on disposable K3s databases. Focused service integration checks and deployment of the fixed runner image are still open. Preliminary archives were captured before quiescence and are not cutover backups. No live database has been modified.

**Why P0:** Each rebuild is destructive and must preserve domain rows, spatial objects, outbox/idempotency state, and consumer checkpoints.

**Next:** Close the owner-mapping and service integration gates, build/deploy the fixed migration image, quiesce each database's writers and async work, take and rehearse a final archive, record go/no-go, then rebuild one database at a time with a tested rollback. Track the exact gates in the [inventory](../operations/databases/baseline-migration-inventory.md) and issues [Deploy #15/#16](https://github.com/myota-platform/myota-deploy/issues/15), [Activity #4/#5](https://github.com/myota-platform/myota-activity-service/issues/4), [Geodata #8/#9](https://github.com/myota-platform/myota-geodata-service/issues/8).

### 2. Reopen Geodata qualification before expanding capacity

**Current position:** The tested deployment envelope is two to three Geodata API
replicas. Phase 5 was closed by owner acceptance, not by passing every
qualification test. Broad-catalogue and sustained-load performance,
independent-row write consistency, node/storage failure, a new-image canary,
end-to-end rollback, and scaling above three replicas remain unverified and
were accepted for closure. The 11 October 2/10/50-VU read-only run used only ten
catalogue entities. See the [owner-acceptance decision](../geodata/evidence/phase5-owner-acceptance-2026-10-11.md),
[read-only results](../geodata/evidence/phase5-read-only-follow-up-2026-10-11.md),
and [scaling roadmap](../geodata/horizontal-scaling-roadmap.md).

**Why P0:** These accepted gaps become release gates again before capacity is
expanded beyond the tested envelope or the stateful topology changes.

**Next:** If expansion is proposed, reopen the relevant roadmap gates and
complete the missing measurements and recovery drills before changing the
replica ceiling. Rerun the digest-pinned Phase 2 recovery qualification when the
deployed SeaweedFS image changes.

### 3. Establish programme governance, jurisdiction, and eligibility controls

**Current position:** The configuration gap analysis remains an audit baseline;
programme ownership/lifecycle, locale and content policy, jurisdiction and
approver scope, structured eligibility, and identity/privacy controls remain
unimplemented roadmap work. See the [gap analysis](../domain/programmes/configuration-gap-analysis.md).

**Why P0:** These policies determine who may administer and review programmes,
which places and activities qualify, and which published rules participants
rely on.

**Next:** Define ownership and audit semantics, stable jurisdiction codes and
scoped approver decisions, versioned eligibility rules, and publication
snapshots. Keep locale/content policy as a dependent but separately scoped
track. Confirm service ownership and authorization boundaries before adding
Admin UI controls.

## P1 — close identified operational visibility gaps

### 4. Complete logging acceptance

**Current position:** Helm revision 272 is deployed and Fleet is Ready with all
23 Deployments and 27/27 replicas. Identity request-to-worker correlation has
passed a sanitized live request, worker completion, and JetStream ACK check;
Identity issue #3 is closed. Identity and Programme metrics coverage is also
complete; see the [metrics evidence](../observability/evidence/identity-programme-metrics-2026-10-11.md).
Remaining logging acceptance includes Programme/Activity/Geodata
producer-specific live correlation evidence, authenticated Grafana log/trace
navigation, a representative Loki prohibited-field/cardinality review, alerts
and runbooks, retention/capacity/recovery evidence, and full Compose runtime
parity. See the [logging roadmap](../observability/logging-implementation.md)
and [current evidence](../observability/evidence/logging-implementation-2026-10-11.md).

**Next:** Close the remaining producer-specific and platform acceptance gates
from their recorded evidence. Do not infer successful workflow correlation for
service paths that have not been verified end to end.

## P2 — planned capability after higher-priority work

### 5. Define the typed future award-condition model

**Current position:** This is a proposed capability. Its roadmap leaves the
semantic and evidence decisions open; later implementation depends on reliable
worked-entity resolution and versioned geodata/reference evidence.

**Next:** Complete the roadmap's Phase 0 evidence and definitions, then decide
which reference-data and QSO-resolution work must precede evaluator
implementation. See the [award-condition roadmap](../domain/awards/future-condition-model-roadmap.md).

### 6. Establish Admin UI internationalization

**Current position:** The proposed roadmap has no completed phases. Locale and
string inventory should precede translation catalogs or screen migration.

**Next:** Choose the source and supported locales, fallback behavior,
persistence, and translation-review ownership. See the
[internationalization roadmap](../domain/administration/i18n-roadmap.md).

### 7. Address longer-term programme and charter gaps

**Current position:** The charter gap analysis is a baseline, not a set of
completed deliverables. Product, governance, interoperability, open-data,
community, and launch gaps remain; programme configuration items classified
P0 are tracked separately above.

**Next:** Split accepted charter findings into deliverable roadmaps with
dependencies, measurable outcomes, and review dates. See the
[charter gap analysis](../governance/charter-gap-analysis.md).

## Reprioritization

- Raise an item when an owner identifies a dated launch, compliance, security,
  or reliability dependency, and record the supporting evidence in its source
  plan.
- Keep rollout gates tied to the capability they qualify. In particular,
  reopen Geodata's accepted qualification gaps before capacity expansion.
- The Work in progress index tracks active delivery; this backlog records
  unfinished work that is not necessarily in progress.
