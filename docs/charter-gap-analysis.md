# Gap analysis against the project charter

Status: **baseline review**  
Reviewed: **2026-10-08**
Inputs: the shared MyOTA project/visibility conversation, the current
repositories, and the implementation documentation in this organization.

This is a product and delivery gap analysis. It does not add rules to MPOTA
or any other programme. “Implemented” means that a meaningful vertical-slice
capability exists in the current repositories; it does not mean that the
capability is production-hardened or publicly launched.

## Summary

The architecture and administration vertical slice remain ahead of the public
participant experience. The participant web now has programme switching,
themed content, an attributed interactive map, entity browsing foundations,
sign-in, and award-level requests. It is still not the complete Explorer and
activation product, and synthetic scale fixtures are not real, licensed park
coverage.

| Charter capability | Current state | Gap / next action | Priority |
| --- | --- | --- | --- |
| Programme-independent platform | Service split, contracts, programme service, sample MPOTA and a second synthetic programme exist | Complete immutable programme publication/versioning and onboarding | P0 |
| Local-park accessibility mission | Geodata review and candidate/approved/rejected workflow exist; permanent synthetic Sevilla scale fixtures are present but are not real parks | Acquire and review real licensed coverage, publish an Explorer with nearby search, and run a measured Sevilla beta | P0 |
| Public participant experience | myota-web has programme switching, theme/content, an attributed interactive map, entity browsing foundations, participant sign-in, and award-level requests | Complete Explorer landing/nearby search and entity details, production account/profile flows, activation/QSO workflows, public history, and privacy-aware leaderboards | P0 |
| Geographic data trust | PostGIS, adapters, provenance, candidate lifecycle, review UI, QGIS guidance, and imports exist | Operate refresh/conflation workers at scale, publish licensing/attribution, and add stewardship workflows | P0 |
| Community verification | Scoped review and audit paths exist | Add evidence, reviewer communication, Park Steward-style local maintenance, SLA and escalation policy | P1 |
| Amateur-radio identity | Accounts, SWL/participant model, callsigns, roles, scopes, OIDC mappings exist | Finish production auth hardening, user self-service, account linking, recovery, privacy, and notifications | P0 |
| Activity execution | Relational activity/QSO schema, ADIF jobs, rule hooks, award definitions, certificate assets and statistics exist | Run realistic load/soak tests, finish public logging flows, and validate correction/recalculation operations | P0 |
| Interoperability | OpenAPI/event contracts and activity/geodata APIs exist | Publish developer portal/examples, stable external-reference endpoints, SDK releases, and logger integration | P1 |
| Open data | Provenance and source licensing metadata are modeled | Decide dataset/API licence, publish approved bulk/change feeds where permitted, and document attribution | P1 |
| Multi-programme policy isolation | Programme-owned policy and awards boundaries are documented and tested with sample data | Finish policy schemas, publication/effective-date governance, jurisdictions, locales, OIDC and simulations | P0 |
| Mobile participant access | Android/iOS user-only roadmap items exist | Define shared mobile API/SDK, offline sync, push, accessibility and implement participant apps | P1 |
| Community launch | No public beta metrics are represented in the codebase | Recruit local operators, run a Sevilla beta, publish evidence, and recruit country coordinators/contributors | P1 |

## Findings from the charter conversation

### 1. The project is not yet externally tangible enough

The repositories demonstrate architecture, admin workflows, and a durable
local stack, but the charter’s strongest public promise is “find a nearby
place and use it.” The public web still needs:

- a map-first Explorer landing page;
- nearby search and entity detail pages;
- clear Candidate / Approved presentation (there is no separate `PROPOSED`
  entity status; community proposals feed the Candidate lifecycle); and
- an end-to-end participant proposal path with authenticated identity and
  review feedback; and
- a first licensed regional dataset large enough to make the experience
  useful.

The permanent Sevilla scale fixtures are synthetic test data, not parks; they
are not evidence of a live community or regional coverage. The participant
client's current capability list and explicit next milestones are maintained
in the [myota-web README](https://github.com/myota-platform/myota-web/blob/main/README.md).

### 2. The governance model is represented technically but not yet socially

The API and admin UI model scoped approvers, audit history, provenance, and
programme-owned policy. The charter also calls for a community that can help
maintain local knowledge. Missing delivery work includes a governance draft,
contributor guidance, a reviewer handbook, escalation rules, a local-steward
workflow, and country/region coordinator onboarding.

### 3. Interoperability is documented but not demonstrated

The contract repository and service endpoints are a good base. The project
still needs public, copyable examples for entities, nearby search, activations,
programmes, external references, and results. A first logger integration or
small import/export tool would prove the “open infrastructure” claim more
effectively than additional internal scaffolding.

### 4. The implementation needs a production-readiness boundary

The local Compose stack is durable, the service split is explicit, and
OpenTelemetry, dashboards, alerts, and production K3s deployment are in place.
Documentation must continue to distinguish “implemented vertical slice” from
“ready for an Internet-facing beta.” Before that beta, complete realistic
scale/soak evidence (including activity/QSO workloads), backup/restore drills,
rate-limit and security validation, and a review of operational controls.

### 5. The public story must remain complementary

The charter conversation explicitly recommends positioning MyOTA as
complementary to existing programmes. Public copy and documentation should
describe the geographic/accessibility problem and open infrastructure without
claiming that another programme’s rules are wrong or importing those rules into
MyOTA defaults.

## Recommended delivery sequence

1. **Make the value visible:** Explorer, nearby search, detail pages, licensed
   Sevilla/Andalucía data, and candidate/approval explanation.
2. **Make participation real:** production user authentication, proposal and
   review feedback, activation logging, ADIF upload, public history, and
   shareable activation results.
3. **Make trust operable:** governance/contributor docs, local stewardship,
   coordinator roles, source refresh/conflation operations, and open-data
   licensing decisions.
4. **Make integration real:** public OpenAPI examples, SDKs, external
   references, and at least one logger/client integration.
5. **Make scale and reach credible:** beta load/security/recovery gates,
   multilingual coverage, and participant-only Android/iOS applications.

## Documentation follow-up

- The organization profile contains the short charter and checkbox roadmap.
- This repository contains the detailed charter and this dated gap analysis.
- programme-configuration-gap-analysis.md remains the authoritative list of
  missing programme configuration fields.
- repository-map.md remains the authoritative ownership and migration rule.
- Service READMEs should describe their bounded responsibility and link back to
  these documents instead of repeating an outdated generic vertical-slice
  description.
