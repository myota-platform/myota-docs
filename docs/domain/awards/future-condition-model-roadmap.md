# Future award-condition model and implementation roadmap

**Status:** proposed future capability; this document does not claim the
conditions or phases below are implemented.

**Progress tracking:** leave items unchecked until evidence is available; mark
`[x]` only when the work is verified. A phase is complete only after all its
exit-gate items are checked and the evidence is recorded.

## Purpose and design constraints

MyOTA awards should support familiar patterns such as outdoor-entity
activations, worked-entity achievements, Maidenhead grids, and geographic
coverage while keeping each award programme's own rules and definitions.
These examples are inspired by common amateur-radio award patterns; they do
not reproduce another programme's rules, entity lists, names, or eligibility
decisions. A programme chooses the catalogues and thresholds it owns or has
permission to use.

The current service already has an award definition with a recursive condition
tree, an achievement metric, configurable levels, hunter/activator categories,
and versioned progress. Its evaluator currently handles boolean composition,
QSO/activation/callsign/entity counts, a coarse entity-type fact, and a generic
numeric `FIELD`. Durable activity data includes activations and normalized QSO
rows; aggregate membership supports participant, programme, callsign, and
entity counts. A QSO can point to `workedEntityId`, but that reference is
optional. Geodata owns authoritative entity geometry, category assignments,
four-character grid-square and six-character locator arrays, plus nullable
continent/country/region/province/county/city names and codes. Award data and
evaluation are Activity-owned; Geodata remains the source of entity and
location evidence.

The future model must not assume that an entity label is a stable identifier,
that an unlinked callsign resolves to a particular country/entity, or that a
missing locator or administrative code means “does not qualify.” Unknown,
ambiguous, unavailable, and ineligible evidence need distinct results. Published
award versions and issued records remain immutable; corrections, entity
changes, catalogue changes, and re-evaluation must leave an audit trail.

## Award patterns to support

| Pattern | Participant activity and evidence | Example progress dimension |
|---|---|---|
| Outdoor entity activation (POTA-like) | Activator completes valid activations from approved programme entities; the award can require a minimum valid QSO count per activation, distinct entity count, date window, or operating constraints. The entity and its programme/category approval come from Geodata; QSOs and activation status come from Activity. | Distinct qualifying entity IDs; optionally activation count or QSO total. |
| Worked entity/country (DXCC-like) | Hunter logs valid QSOs linked to a canonical worked entity. A separately governed entity-resolution catalogue may map a resolved station/entity to a country or award entity. Callsign-prefix guesses alone are not authoritative award evidence. | Distinct canonical entity codes or countries; optional band/mode endorsement buckets. |
| Maidenhead grids | Activator credit derives from the activation entity's grid coverage; hunter credit derives from the worked entity or a captured worked-station locator. The award must explicitly choose centroid-cell versus any-intersected-cell credit. | Distinct 4-character squares or 6-character locators. |
| Geographic coverage | Activator or hunter credit uses versioned geographic identifiers for country, first-level subdivision (state/province/region), county/district, or programme-defined area. | Distinct countries, subdivisions, counties, or nested combinations. |
| Operating milestones | Valid QSOs/activations constrained by UTC range, band, mode, callsign class, power, portable status, or verified programme metadata. Only fields with a defined, trusted source may be used. | Count, distinct entities, or one bucket per band/mode/period. |
| MyOTA additions | A progression ladder can reward entity diversity, geographic breadth, mixed entity categories, verified stewardship/community actions, or challenge seasons. An endorsement can share the same evidence with a narrower filter (for example, one band or one mode). Future non-QSO evidence must be typed, attributable, reviewable, and programme-owned. | Level, endorsement, diversity score, or season-specific progress. |

These patterns combine through conditions rather than hard-coded award types.
The designer may offer named templates (“activate ten entities,” “work 100
resolved entities,” “cover five grid squares”) that compile to the same typed
condition format.

## Proposed condition structure

Keep the persisted award definition versioned and programme-owned. Replace
free-form numeric `FIELD` lookups for new definitions with a validated,
allowlisted condition schema. Continue reading older version-1 conditions and
preserve their meaning during migration.

### Definition sections

- **`conditionSchemaVersion`** — version of the evaluator grammar, independent
  of the award definition version.
- **`category`** — `ACTIVATOR` or `HUNTER`, as today; later categories require
  an explicit evidence source and authorization policy.
- **`eligibility`** — a typed boolean condition tree over typed evidence.
- **`progress`** — the metric and dimensions displayed to participants, such
  as distinct entities, distinct grids, or qualifying activations.
- **`levels`** — stable IDs, labels, thresholds, optional effective periods,
  and optional endorsement qualifiers. Thresholds must be monotonic where the
  progression is cumulative.
- **`evidencePolicy`** — required source/status, missing-data behavior, time
  basis, reference-catalogue versions, and recalculation policy.

An evidence query has a bounded `source`, optional `where` predicates, an
optional `distinctBy` key, a typed `measure`, and threshold comparison. Boolean
nodes combine these queries. Keep the AST declarative and bounded: no embedded
SQL, arbitrary JSON paths, executable expressions, remote lookups during each
participant read, or unbounded nested recursion.

Suggested initial query fields:

| Evidence source | Candidate fields | Source and current coverage |
|---|---|---|
| `ACTIVATION` | `id`, `status`, `startedAt`, `endedAt`, `entityId`, `entityCategoryCodes`, `entityType`, `jurisdictionCode`, `validQsoCount`, `bandSet`, `modeSet` | Activity activation and QSO records; entity enrichment/assignments resolve through Geodata. `jurisdictionCode` exists but does not yet mean a universal country/state/county identity. |
| `QSO` | `id`, `status`, `occurredAt`, `band`, `mode`, `workedCallsign`, `workedStationKey`, `workedEntityId`, `workedEntityCode`, `workedCountryCode`, `workedSubdivisionCode`, `workedCountyCode`, `workedGrid4`, `workedGrid6` | Activity QSO rows provide the first fields and optional `workedEntityId`; canonical worked-station geography/grid resolution is a future data-coverage task. |
| `ENTITY` | `id`, `categoryCode`, `lifecycleStatus`, `countryCode`, `subdivisionCode`, `countyCode`, `grid4Cells`, `grid6Cells`, `entityTypeCode` | Geodata entity catalogue, geometry-derived Maidenhead fields, and programme category assignments. Names are display values; award matching uses stable codes/IDs. |

`ENTITY` is joined only through an explicit activation or QSO entity reference.
An entity's category assignment must be scoped correctly: a shared category
catalogue code is not proof that an entity is approved for a specific
programme. Candidate/rejected/retired entity states must follow a published
policy; the default should credit only entities and activities valid under the
award's effective rules.

### Condition grammar proposal

```json
{
  "conditionSchemaVersion": 2,
  "eligibility": {
    "kind": "ALL",
    "conditions": [
      {
        "kind": "COUNT",
        "source": "ACTIVATION",
        "where": [
          { "field": "status", "op": "EQ", "value": "CLOSED" },
          { "field": "validQsoCount", "op": "GTE", "value": 10 },
          {
            "field": "entity.categoryCode",
            "op": "IN",
            "values": ["PARK"]
          }
        ],
        "distinctBy": "entityId",
        "op": "GTE",
        "value": 10
      },
      {
        "kind": "DATE_RANGE",
        "source": "ACTIVATION",
        "field": "startedAt",
        "from": "2027-01-01T00:00:00Z",
        "toExclusive": "2028-01-01T00:00:00Z"
      }
    ]
  },
  "progress": {
    "measure": "DISTINCT_COUNT",
    "source": "ACTIVATION",
    "field": "entityId",
    "filters": [{ "field": "validQsoCount", "op": "GTE", "value": 10 }]
  }
}
```

The example is illustrative. The schema must define whether the date range
applies to each counted activation or to the participant's overall qualifying
set, whether a per-activation QSO minimum uses valid QSO rows at the close
snapshot, and how a later reviewed correction affects historic qualification.
The evaluator should compile these semantics to Activity-owned queries and
explicit Geodata/reference lookups; it should never trust client-submitted facts
for participant award claims.

Additional typed predicates can be added without changing the core query shape:

- `IN` / `NOT_IN` against versioned code lists;
- `DATE_RANGE` with an inclusive start and exclusive end in UTC;
- `ALL_OF`, `ANY_OF`, and `NONE_OF` across entity/grid/geographic sets;
- `BUCKETED_COUNT` grouped by a bounded dimension such as band, mode, year,
  country, subdivision, grid precision, or entity category;
- `PER_ITEM` for constraints such as a minimum QSO count per activation;
- `SEQUENCE` or `STREAK` only in a later phase with a clearly defined calendar
  and missing-period policy.

Do not make a raw `FIELD` clause able to reference any key in JSON. Validation
must resolve each `source.field` against a versioned registry, enforce type,
operator and list-size constraints, reject unknown fields/operators, and cap
AST depth, number of nodes, date span, and dimension cardinality.

### Worked examples

**Activator: ten qualifying entities with ten valid QSOs each**

Count distinct `ACTIVATION.entityId` where the activation belongs to the
participant, is closed and valid, the entity is approved for the selected
programme/category, and at least ten non-void QSOs belong to that activation.
Levels can be 1, 10, and 25 entities. A repeated activation at the same entity
adds activity but not another distinct entity.

**Hunter: 100 resolved award entities**

Count distinct `QSO.workedEntityCode` for valid hunter QSOs with a reviewed,
versioned entity resolution. Unresolved callsigns contribute to QSO totals but
not to this award measure. An endorsement can require a minimum number of
entities on a selected band or mode.

**Grid award: 50 four-character squares**

Count distinct grid codes at precision 4 from the chosen evidence source. For
activation entities, choose whether credit uses the geometry centroid or every
intersected grid cell. The all-intersected policy can grant many squares for a
large park/line/polygon; show that implication in the designer and cap or review
unusually large credit sets. For hunter QSOs, prefer a validated worked
station-locator captured in the QSO or an explicit entity association; never
infer a grid from the hunter's own activation.

**Geographic award: five countries and ten subdivisions**

Count distinct ISO country codes and distinct subdivision codes from an
approved, versioned geographic reference set. County/district awards need a
dataset-specific stable code and country context, for example a composite
`referenceSet + countryCode + areaCode`; a county name alone is ambiguous.
Require administrators to choose one code system and coverage version for the
award.

**MyOTA seasonal diversity challenge**

Within a UTC season, require a threshold of qualifying contacts plus at least
three bands, two modes, and five distinct geographic areas. Store bucketed
progress so the participant can see exactly which requirement remains. This is
a candidate future template, not an assumed platform-wide rule.

## Data, provenance, and evaluation rules

1. **Activity remains award authority.** Activity owns QSO/activation status,
   subject identity, aggregation, condition evaluation, progress, requests, and
   issuance. Geodata is queried through a bounded service contract or a
   versioned projection; Activity must not read Geodata tables directly.
2. **Entity references are stable.** Award evidence stores stable entity IDs and
   reference codes plus the source/version used for resolution. Names and
   mutable labels are presentation only.
3. **Resolution is explicit.** `workedEntityId` is optional today. A future
   resolver must distinguish `RESOLVED`, `AMBIGUOUS`, `UNRESOLVED`, and
   `REVIEW_REQUIRED`; resolved codes need source, confidence/review status, and
   effective time. Callsign prefixes can propose candidates but cannot silently
   grant DXCC-style credit.
4. **Geography is versioned.** Persist code system, reference-set version, and
   geographic level. ISO country, state/province, region, and county/district
   identifiers differ by source and jurisdiction; never equate unlike codes or
   fall back to display-name equality.
5. **Grid credit is a named policy.** Geodata stores all four-character and
   six-character cells intersecting an entity geometry. Awards choose
   `CENTROID_CELL` or `ANY_INTERSECTED_CELL`, precision, and entity coverage
   version. For hunter credit, record the worked locator or evidence link
   rather than reusing the operator's activation entity locator.
6. **Missing is not false.** Evaluation exposes `QUALIFIED`, `NOT_QUALIFIED`,
   `PENDING_EVIDENCE`, and `REVIEW_REQUIRED`. Missing QSO links or geographic
   reference coverage cannot be silently treated as a non-credit or as a
   credit. Participant-facing explanations identify unavailable evidence.
7. **Corrections trigger controlled recomputation.** QSO void/correction,
   activation invalidation, entity deletion/category change, reference
   resolution, and condition-version changes enqueue bounded recalculation.
   Idempotent jobs replace progress but never rewrite an issued certificate or
   silently revoke an issuance. Programme policy decides how already issued
   awards are handled.
8. **Definitions and evidence are reproducible.** Published awards snapshot
   condition schema version, award version, effective period, reference/grid
   versions, and measurement semantics. Progress records the computed time,
   input revision/cursor, evidence summary, unresolved count, and rule version.
   Award requests and immutable issuance preserve the qualifying snapshot.
9. **Scale is bounded.** Participant progress uses indexed SQL and
   materialized aggregates; no participant read scans every QSO. Recalculation
   can target one subject, award, period, or affected entity/geographic bucket.
   Cross-service lookups are paged/batched and cached as versioned evidence,
   never done once per QSO in a full table scan.

## Phased implementation plan

### Phase 0 — Evidence inventory and final semantics

**Deliverables**

- [ ] Map each existing condition kind, evaluator fact, award level, and progress
      query to the durable Activity schema and existing Admin UI.
- [ ] Inventory how `workedEntityId` is populated, which QSO import/adjudication
      paths can set it, and the share of historical valid QSOs with no link.
- [ ] Verify what `jurisdictionCode` means in each producer and which Geo fields
      have stable codes versus names only. Confirm grid arrays' calculation and
      credit semantics for points, lines, and polygons.
- [ ] Define candidate reference-data owners and licensing/update rules for
      worked-entity resolution and country/subdivision/county code sets.
- [ ] Agree `conditionSchemaVersion`, condition/query semantics, unknown-evidence
      states, award effective-time behavior, correction policy, and migration
      compatibility. Record these as an ADR/API contract before implementing.

**Exit gate**

- [ ] Examples above have unambiguous expected evidence and results; no
      implementation assumes uncovered historical data is complete.


**ChatGPT prompt — Phase 0**

```text
Work in the MyOTA multi-repository workspace. This phase is analysis and docs only:
do not change runtime code or claim any proposed award feature is implemented.

Inspect myota-activity-service award evaluation, aggregate/progress tables, QSO and
activation schemas, API contracts, test fixtures and Admin award designer. Inspect
myota-geodata-service entity/category/location/grid schemas and APIs, plus
myota-contracts and the relevant MyOTA docs. Treat service-owned source as
authoritative and deploy/platform copies as synchronized mirrors.

Produce an evidence-backed gap matrix for current conditions and each proposed
pattern: outdoor entity activations, worked canonical entities, Maidenhead grids,
country/subdivision/county coverage, operating filters, and seasonal or diversity
awards. Trace exactly which fields are durable today, which are optional/null, how
each is populated, who owns it, and whether historical rows can be backfilled
safely. Investigate workedEntityId coverage and do not infer a worked entity or
locator from a callsign prefix without a governed resolution. Explain the grid
credit choice for entity geometries (centroid versus all intersected cells), and
identify jurisdiction-code ambiguities and reference data/licensing owners.

Edit only myota-docs: refine docs/domain/awards/future-condition-model-roadmap.md with
findings, assumptions, and unresolved decisions; link it from docs/README.md in the
domain/API section. If you find conflicting current claims, cite paths and propose a
correction without editing service code. Do not mark implementation phases complete.
Return the evidence matrix and the specific decisions needed before Phase 1.
```

### Phase 1 — Versioned reference evidence and contract

**Deliverables**

- [ ] Define versioned reference sets for award entities and nested geography.
      Select the data owner, code systems, effective dates, provenance, update and
      licensing process; preserve prior versions needed by published awards.
- [ ] Define the worked-station resolution lifecycle and QSO evidence fields.
      Review/import tooling should show source and confidence and support human
      resolution for ambiguous matches. Do not silently rewrite existing QSOs.
- [ ] Define typed condition JSON schema, field/operator registry, AST bounds,
      `conditionSchemaVersion`, deterministic semantics and compatibility rules.
- [ ] Define Geodata APIs/projections for programme-approved entity categories,
      stable codes, grid coverage, and geographic reference version. Ensure API
      responses are paged/batched and include an explicit data version.
- [ ] Keep old condition definitions readable. New version publishing must capture
      exact evaluator and reference versions used for qualification.

**Exit gate**

- [ ] Contracts and provenance are reviewed; example awards validate and explain
      their result from immutable fixtures.


**ChatGPT prompt — Phase 1**

```text
Implement Phase 1 of the award-condition roadmap only after its Phase 0 decisions
are recorded. First inspect the agreed ADR/contract and current source owners. If
reference-data ownership, licensing, condition semantics, or missing- evidence
behavior is unresolved, stop schema/runtime changes and document the decision
required.

In myota-contracts, define the versioned award-condition schema and typed
evidence/reference contracts. In the owning service(s), add only the agreed
versioned reference-data and worked-station-resolution model/API. Preserve existing
QSO rows and condition v1 behavior; do not auto-assign an award entity from a
callsign prefix. Store source, code system, reference-set version, effective time,
confidence/review state, and stable IDs. Define bounded batch lookup and pagination
between Activity and Geodata; Activity must not query Geodata tables directly.
Include grid-credit policy (centroid or intersected-cell), precision, geometry
version, and documented geographic code hierarchy.

Update canonical OpenAPI/schema/docs first, then synchronize platform/deploy mirrors
from their source owners. Add fixture definitions for an entity- activation award,
resolved-entity hunter award, grid award, and nested geography award. Validate
schema bounds and reject unknown source fields, operators, code systems, or
unreviewed evidence where the contract requires it. Add focused contract tests only
in the existing relevant test suites. Report repo/file ownership, migration/backfill
safety, and any unresolved source data. Do not begin aggregate evaluation or Admin
UI work in this phase.
```

### Phase 2 — Evaluator, progress facts, and recomputation

**Deliverables**

- [ ] Add a typed evaluator/query compiler for approved `COUNT`, distinct-count,
      date-range, set membership, per-item, and bounded bucket conditions.
- [ ] Make evaluator reads operate on `VALID`/`CORRECTED` evidence according to
      explicit policy; exclude `VOID` QSOs. Keep activation validity, minimum QSO,
      entity approval, award dates, and participant role semantics explicit.
- [ ] Extend Activity-owned materialized aggregates or add bounded progress tables
      for entity/geography/grid/band/mode/year buckets. Include indexes and rebuild
      cursor/job state; avoid full scans on interactive participant reads.
- [ ] Add explainable evaluation results with status, requirement-by-requirement
      progress, counted IDs/codes or bounded evidence summaries, unresolved/missing
      counts, rule/reference versions, and computation cursor.
- [ ] Trigger idempotent recalculation for QSO ingestion/correction, activation
      close/invalidity, linked entity change/deletion, resolver review, and award
      publication. Define how previously issued awards are treated separately.
- [ ] Keep explicit-facts evaluation restricted to trusted internal/admin paths;
      participant evaluation must derive from persisted facts.

**Exit gate**

- [ ] Reference fixtures, correction/deletion/replay scenarios, and performance
      evidence prove stable progress without scanning all QSOs per read.


**ChatGPT prompt — Phase 2**

```text
Implement Phase 2 of the award-condition roadmap using the approved Phase 1
contract. Work in myota-activity-service and its authoritative schema; call Geodata
only through the contracted bounded API/projection. Keep legacy award condition v1
readable and deterministic.

Build a validator and evaluator for the approved typed AST, including numeric
counts, distinct count by stable ID/code, typed filters, UTC date ranges, per-item
constraints and bounded bucketed progress. Compile only allowlisted
fields/operators; enforce AST depth/node/list/date-span and query/concurrency
bounds. Make unknown/unresolved evidence produce the contracted pending/review
state, never a silent credit or guessed geography. Use persisted participant
identity and QSO/activation data; do not accept client facts for participant
qualification. Exclude void QSOs and apply approved correction/activation validity
rules.

Add the Activity-owned aggregate/progress storage and indexes chosen in Phase 1.
Recalculation must target affected subjects/awards/periods and be idempotent,
restartable and observable; participant progress reads must not scan the full QSO
table. Store evaluator/award/reference versions, cursor, explainable per-requirement
progress and bounded evidence provenance. Wire recalculation to relevant QSO,
activation, entity, resolver and award lifecycle changes. Do not rewrite immutable
issuance records; implement only the approved policy for old issuance after later
corrections.

Add focused evaluator, SQL, migration, idempotency and correction/deletion tests
following existing conventions. Synchronize deploy/platform copies from the owning
source and update contracts/docs. Report query plans or bounded performance
evidence, compatibility behavior and unresolved evidence coverage. Do not implement
the Admin builder in this phase.
```

### Phase 3 — Admin award designer and preview

**Deliverables**

- [ ] Replace raw JSON editing as the normal path with a guided condition builder;
      retain a guarded read-only/advanced JSON view only if it validates against the
      exact public schema.
- [ ] Builder sections: award participant category; evidence source; eligible entity
      categories/programme assignments; condition metric; filters; distinct
      dimension; thresholds/levels; validity dates; geography/grid policy; missing
      evidence policy; endorsements.
- [ ] Provide searchable code selectors for programme entities, reference sets, grid
      precision, country/subdivision/county and band/mode. Show source/version and
      coverage before publication; no free-text code matching.
- [ ] Add a server-side preview against a selected participant or synthetic fixture.
      Display qualified/not qualified/pending/review-required, per-level status, the
      evidence counted, exclusions, and unresolved data. Clearly label preview as
      non-issuance and prevent changes to published definitions.
- [ ] Add complexity warnings (large geometry credit, broad date span, many buckets,
      unsupported evidence coverage) and accessible keyboard/screen-reader controls.
      Preserve draft metadata, saved condition version and UTC instants.

**Exit gate**

- [ ] Admin can author and preview all four core pattern types without hand-editing
      JSON; server validation remains authoritative.


**ChatGPT prompt — Phase 3**

```text
Implement Phase 3 of the award-condition roadmap in myota-admin-web, using the
published condition schema and evaluator preview API from Phase 1/2. Inspect the
existing AwardsView.vue and browser tests; preserve the current artwork, certificate
layout, programme selection, draft lifecycle and metadata-safe save behavior.

Replace raw JSON as the default authoring interface with a guided accessible builder
for hunter/activator scope, evidence source, eligible programme entity categories,
count/distinct metric, filters, levels, date range, grid precision and credit
policy, and versioned country/subdivision/county references. Provide POTA-like
entity activation, resolved-entity hunter, grid, nested geography and
season/diversity templates that compile to the same typed AST. Use search APIs for
stable IDs/codes and display reference source/version and coverage; do not accept
free-text geographic matching. Preserve legacy v1 awards through an explicit
read/edit migration path and keep published versions immutable.

Integrate a server-side preview that shows each requirement, progress, qualifying
evidence summary, exclusions and pending/unresolved state. Preview must never issue
an award or save draft state. Warn about incomplete QSO resolution, geometry
multi-cell credit, broad/high-cost filters and unavailable reference coverage before
publish. Validate the condition server-side and show clear path-specific errors.
Keep UTC effective-date behavior, keyboard operation, responsive layout, and
screen-reader labels.

Update API client types only from canonical contracts, add focused browser tests for
builder round trips, template composition, preview states, invalid schemas, legacy
drafts and published read-only behavior, and update docs for the final UI. Do not
add evaluator semantics that the server does not support. Report accessibility and
integration evidence and any deferred controls.
```

### Phase 4 — Participant progress, endorsements, and award history

**Deliverables**

- [ ] Extend public programme award pages with transparent requirement progress,
      levels, endorsement buckets, evidence freshness, and pending resolver/review
      work. Respect callsign/account privacy settings and authorization.
- [ ] Add participant-request confirmation that snapshots the qualifying award,
      condition/reference versions and progress. Keep manager review and issuance
      separate where the programme requires it.
- [ ] Make corrections or reference revisions visible in progress history. Never
      silently alter an issued certificate; surface programme policy for revocation
      or replacement as an explicit auditable workflow if one is adopted.
- [ ] Backfill only reliable historical facts; produce coverage reports before
      enabling geographic or DXCC-like awards. Provide appeal/manual-review paths
      with evidence and actor audit.

**Exit gate**

- [ ] Participant-facing explanations match server evaluation; privacy, correction,
      manual-review, and immutable-issuance behavior are accepted by programme
      governance.


**ChatGPT prompt — Phase 4**

```text
Implement Phase 4 of the award-condition roadmap across the participant web, Admin
web where review is required, Activity APIs and contracts. Read the approved
participant progress/appeal policy and Phase 1-3 completion evidence before changing
any issuance flow.

Present per-requirement and per-level progress from the server evaluator, including
distinct entity/grid/geography buckets, reference/evidence freshness, and pending or
review-required records. Do not recalculate eligibility in the browser. Respect
hunter/activator identity, private callsign/account data and public masking rules. A
request must bind to a specific immutable award version and qualifying snapshot;
issuance remains an explicit programme-authorized action. Preserve the existing
certificate issuance record and rendering flow.

If approved policy introduces appeal, manual adjudication, revocation or
replacement, implement explicit audited states and actor/reason/evidence links; do
not silently mutate an issued certificate when a QSO is corrected or a reference set
changes. Add historical-coverage reports before enabling awards that depend on
optional worked-entity/grid/geography fields. Backfill only reliable evidence and
mark incomplete history as pending/reviewable.

Add focused API/browser tests for privacy, stale snapshots, pending resolution,
manual review, correction after issue, and request idempotency. Update
participant/Admin docs and programme-facing templates. Report exact backfill
coverage, authorization review and any policy decision that still blocks
publication.
```

### Phase 5 — Pilot, qualification, and broader award catalogue

**Deliverables**

- [ ] Pilot each pattern with synthetic fixtures and a volunteer programme before
      opening general award authoring. Measure QSO ingest, recalculation duration,
      backlog, API latency, aggregate growth, and Geo reference coverage.
- [ ] Reconcile progress against direct bounded database queries and manually
      reviewed samples; test duplicate imports, correction, void, deletion, changed
      entity assignment, late reference resolution, and replay.
- [ ] Document reference-set refresh, retention, availability, stale projections,
      incident response, programme migration, and rollback.
- [ ] Add new condition types (streaks, seasons, distance, satellite, portable/QRP,
      verified stewardship) only when source evidence, validation, UI, privacy, and
      recalculation semantics are separately agreed.

**Exit gate**

- [ ] Each enabled condition type has programme-approved rules, provenance and
      coverage, performance/recovery evidence, a supportable operator runbook, and
      no unresolved silent-credit behavior.


**ChatGPT prompt — Phase 5**

```text
Run Phase 5 qualification for the award-condition implementation. This is an
evidence and controlled-pilot phase; do not broaden production eligibility or enable
new awards until programme owners accept the coverage and behavior.

Create synthetic fixture cohorts for activator/entity, hunter/resolved entity,
Maidenhead, country/subdivision/county, and seasonal/diversity patterns. Include
ambiguous/unresolved identities, missing codes, multi-cell geometries, boundary
dates, void/corrected QSOs, duplicate imports, changed entity category, entity
deletion, late resolution, and reference-set version changes. Compare evaluator
results with independently calculated expected results and a reviewed sample.
Measure bounded progress reads, recalculation throughput and backlog, aggregate
growth, worker restart/replay behavior, and API latency at the agreed target data
size. Separate synthetic evidence from live user/data coverage.

Pilot with one consenting programme and documented eligibility definitions.
Reconcile participant progress and requests before/after each correction and
reference update. Verify audit, appeal/manual review, immutable issued records,
privacy masking, monitoring/alerts, reference freshness, retention and rollback.
Update this roadmap with evidence, limitations, responsible owners and explicit
go/no-go decisions. Do not mark a condition type complete because its UI or unit
tests exist; require programme sign-off and production-like recovery and capacity
evidence.

Only after sign-off, prepare an independently reviewed rollout for additional
programmes. Propose new types such as streaks, distance, satellite, QRP/portable or
stewardship in separate decisions that name authoritative evidence and
privacy/recalculation costs. Report what can be enabled safely and the exact
remaining gaps.
```

## Administration UI work summary

The future UI change is a substantive award-authoring feature, not just a new
text field. Admins need to select the participant role and evidence source,
choose a familiar template, define typed filters and unique dimensions, create
levels/endorsements, select data/reference versions, preview evaluated evidence,
understand unknown coverage, and preserve immutable published definitions.
Award templates are conveniences over the same validated schema; the server
remains the source of truth. Keep previews non-mutating, do not let the browser
submit trusted facts, and do not expose raw personal QSO evidence beyond the
authorizer's scope.

## Completion and change control

Track phase status and evidence here. Keep service schemas and code in their
owning repositories, contracts in `myota-contracts`, Admin UI work in
`myota-admin-web`, and synchronized runtime/deployment mirrors updated through
their owners. Update [Awards and programme execution](execution.md),
[the programme and award designer guide](admin-designer.md),
contracts, and the repository map when implemented behavior changes. A checked
phase means its exit gate is evidenced; it does not claim every possible award
pattern is available.
