# Programme configuration gap analysis

Status: **audit baseline — implementation pending**  
Owners: `myota-programme-service` and `myota-admin-web`, with configuration
boundaries shared with identity, geodata, activity, and deployment.

This document records what is still missing from programme configuration after
the current data-model and general-administration work. It is deliberately a
gap list, not a platform-wide backlog. Each programme must define its own
charter, eligibility, rules, awards, moderation policy, and public content;
the platform must not copy or impose rules from MPOTA, POTA, or another
programme.

## What is already available

The current vertical slice already provides these foundations:

| Area | Current capability | Owner |
| --- | --- | --- |
| Programme record | Create, edit, list, and archive programmes; name, slug, description, theme colors, and status | `myota-programme-service` |
| Shared categories | Database-backed entity-category catalogue; a category can be assigned to several programmes; an entity can have several categories | `myota-programme-service`, `myota-geodata-service` |
| Geometry policy hint | Category assignments can describe accepted Point, LineString, MultiLineString, Polygon, and MultiPolygon geometry types | Programme and geodata services |
| Common rules | Minimum activation/hunter QSOs, finite or unlimited activation validity, public-access flag, and overlap flag | Programme service |
| Policy drafts | Rule and award policy drafts can be submitted, reviewed, published, and given an effective date | Programme service; award execution is in activity |
| Content workflow | Localized content drafts, review/publish states, fallback locale field, and coverage reporting | Programme service and admin web |
| Award execution | Programme-linked hunter/activator awards, conditions, levels, assets, signatures, requests, and issuances | `myota-activity-service` |
| Identity primitives | Accounts, callsigns, roles, scoped role assignments, and stored per-programme OIDC provider mappings | `myota-identity-service` |
| Geodata lifecycle | Programme-independent imports and provenance; candidate, proposed, approved, retired, and rejected review states | `myota-geodata-service` |
| Activity execution | Activation/QSO primitives, idempotency, callsign checks, time/band/mode inputs, and rule-evaluation hooks | `myota-activity-service` |

These capabilities are useful primitives, but several are still exposed as
generic JSON or service APIs rather than as a complete, validated programme
configuration experience.

## Missing or incomplete configuration

### 1. Programme identity, ownership, and governance — P0

- [ ] Add structured owner organisation/person, programme contacts, public
  website, logo/icon/banner assets, short/public descriptions, and public
  links.
- [ ] Add legal/charter, terms, privacy, contact, and data-licence links with
  effective dates and visible public attribution.
- [ ] Add a programme lifecycle with draft, under-review, published/active,
  suspended, retired, and archived states. Require an explicit publication
  decision and effective date; do not treat create/update as publication.
- [ ] Add immutable configuration versions, supersession links, retirement
  reason, and a historical snapshot used by activity and award evaluation.
- [ ] Add ownership, delegated-administrator, approver, moderator, and
  escalation contacts with an auditable change history.
- [ ] Add a programme timezone and default country/region where relevant to
  local dates and operator-facing displays.

### 2. Locales and programme content — P0

- [ ] Add a programme-owned locale catalogue with enabled locales, default
  locale, ordered fallback chain, and locale retirement behavior.
- [ ] Add required content keys/namespaces and validation for missing,
  malformed, or unpublished translations.
- [ ] Add translation import/export, content version comparison, and a
  programme-owner sign-off step for publication.
- [ ] Add rich-content policy (plain text/Markdown/HTML), link validation,
  accessibility checks, and an explicit missing-translation fallback policy.
- [ ] Report coverage per locale against the programme's required key set;
  the current coverage view is not yet backed by a complete locale catalogue.

### 3. Theme and public presentation — P1

- [ ] Extend theme configuration beyond primary/accent colors to include
  accessible surface/text colors, contrast validation, typography choices,
  logos, map/status colors, and light/dark behavior.
- [ ] Add image asset storage, resizing/cropping rules, attribution, and
  programme-specific public navigation/landing-page configuration.
- [ ] Add an accessibility preview and reject combinations that fail the
  programme's configured contrast target.

### 4. Jurisdictions and approver scopes — P0

- [ ] Create a programme-owned jurisdiction catalogue with stable codes,
  names, hierarchy, country/region mapping, aliases, and optional boundary
  geometries.
- [ ] Define whether jurisdictions may overlap and how an entity is assigned
  when boundaries or source metadata conflict.
- [ ] Add programme + jurisdiction + category approver-scope management to the
  admin UI, including a way to test which reviewer can act on a queue item.
- [ ] Support jurisdiction-specific rules, awards, content, and moderation
  policy without duplicating the whole programme.

### 5. Structured eligibility and activation policy — P0

- [ ] Replace the editor's small set of common controls and opaque JSON with a
  versioned, programme-owned rule schema and schema validation.
- [ ] Configure eligible entity statuses and category codes, including the
  rule that approved entities remain historically valid and may only move to
  retired status.
- [ ] Configure geometry constraints, required activation location, allowed
  distance/buffer from the entity, coordinate accuracy, and location evidence.
- [ ] Configure activation duration, maximum hours, validity windows, UTC/local
  time behavior, start/close state transitions, and invalidation/review rules.
- [ ] Configure operator versus SWL participation, verified-callsign
  requirements, multiple-callsign selection, and whether unverified/guest
  participation is allowed.
- [ ] Configure accepted bands and modes, QSO time-window behavior, duplicate
  and correction policy, minimum/maximum QSO counts, and evidence/photo
  requirements.
- [ ] Replace the current overlap boolean with explicit cross-programme
  behavior, precedence, and conflict handling.
- [ ] Add policy version/effective-date management, retirement semantics,
  historical snapshots, a dry-run evaluator, and sample activation/QSO
  simulation before publication.

### 6. Awards and programme policy linkage — P1

Concrete award definitions, artwork, signatures, conditions, levels, requests,
and certificates belong to `myota-activity-service`; they must not be copied
into the programme service. Programme configuration still needs:

- [ ] A programme-level link/summary of published awards owned by activity,
  including award-rule version and effective date.
- [ ] Allowed award categories and participant types (hunter, activator, or
  both), qualification source, and programme-specific eligibility switches.
- [ ] Certificate issuance, manager-approval, re-request, higher-level, and
  public-visibility defaults.
- [ ] Programme default award manager/signature policy without overwriting the
  activity service's asset ownership.

### 7. Geodata eligibility and moderation policy — P1

Imports intentionally remain programme-independent. The missing programme-side
configuration is the policy that determines how shared geodata may be used:

- [ ] Accepted category assignments, entity statuses, geometry types, and
  source/provenance/licence requirements.
- [ ] Jurisdiction acceptance, source freshness, duplicate/conflation
  thresholds, and disappearance/retirement semantics.
- [ ] Candidate/proposed/approved transition requirements, approver scope,
  review SLA, evidence requirements, and rejection/reconsideration policy.
- [ ] Multiple-category semantics, including whether a primary category is
  required for a legacy integration and how category changes affect rules.
- [ ] Programme-facing public map defaults and status colors while keeping
  tile-provider and deployment settings in platform/deployment configuration.

### 8. Identity, OIDC, and privacy policy — P0/P1

- [ ] Expose the existing per-programme OIDC mapping in Programme Management:
  issuer, client ID, requested scopes, claim mapping, allowed domains,
  enable/disable, and connection-test status.
- [ ] Configure external-login button/redirect policy and the account-linking
  behavior for existing callsigns; provider secrets must remain in secret
  management, not programme JSON.
- [ ] Link programme roles and scoped permissions to the programme's owner,
  approver, moderator, activity manager, and read-only roles.
- [ ] Configure public callsign masking/display, public profile fields,
  leaderboard visibility, retention, export, correction, and deletion policy.
- [ ] Configure security-event visibility and notification requirements for
  programme administrators.

### 9. Notifications, public outputs, and operations — P1/P2

- [ ] Configure programme notification channels, sender identity, templates,
  locale variants, and subscriptions for proposal decisions, import failures,
  award qualification, and account-security events.
- [ ] Configure public activation-history, leaderboard, downloadable-results,
  and callsign-privacy defaults.
- [ ] Configure statistics periods, timezone, reproducible recalculation
  policy, publication delay, and export retention.
- [ ] Allow a programme to reference approved geodata sources and attribution
  text without making the import pipeline programme-bound.

## Ownership boundaries

| Concern | Owning repository | Programme configuration responsibility |
| --- | --- | --- |
| Programme metadata, versions, policy, locales, jurisdictions, category assignments | `myota-programme-service` | Store and publish programme-owned configuration and immutable versions |
| Accounts, callsigns, roles, scopes, OIDC mappings | `myota-identity-service` | Provide programme policy inputs and delegated-admin bindings |
| Entities, geometries, imports, provenance, review/audit | `myota-geodata-service` | Enforce programme eligibility/moderation policy when a programme uses an entity |
| Activations, QSOs, execution, awards, certificates, statistics | `myota-activity-service` | Evaluate the published programme snapshot and retain the rule version used |
| Admin forms and review workflows | `myota-admin-web` | Expose validated configuration, previews, publication, and audit history |
| Runtime topology, secrets, tiles, object storage, workers | `myota-deploy` / platform | Supply operational settings; do not become programme rule storage |

## Recommended implementation order

1. Programme lifecycle, governance metadata, immutable versions, and
   publication/effective dates.
2. Locale catalogue/default/fallback and structured policy schemas with
   validation and a dry-run simulator.
3. Jurisdiction boundaries and approver-scope management.
4. Complete activation/QSO, geodata eligibility, identity/OIDC, privacy, and
   public-output policy forms.
5. Award linkage, notifications, presentation assets, translation tooling,
   and advanced analytics defaults.

## Non-goals and design constraints

- Do not copy POTA, MPOTA, or another programme's rules, charter, award
  thresholds, or eligibility assumptions into platform defaults.
- Do not move concrete award definitions or certificate assets into the
  programme service; activity owns award execution.
- Do not make geodata import programme-bound; imports produce shared candidate
  data and programmes decide which data they accept.
- Do not store provider secrets, signing keys, or operational credentials in
  programme configuration JSON.
