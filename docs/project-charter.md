# MyOTA purpose, motivation, and charter

Status: **working platform charter**  
Reviewed: **2026-09-27**

This document records the project purpose and positioning agreed during the
MyOTA planning conversation. It describes the platform and its governance
boundaries; it does not define the rules of MPOTA or any other programme.

## Purpose

MyOTA is open infrastructure for geographic amateur-radio activation
programmes. It lets an organisation or community configure its own programme
on a shared platform for:

- discovering geographic entities;
- proposing, validating, and maintaining authoritative locations;
- starting and recording activations and QSOs;
- publishing programme-owned awards and statistics; and
- exposing stable APIs and data for radio, mapping, and logging tools.

MyOTA is the platform and ecosystem. MPOTA is retained as synthetic sample
data and a possible first programme configuration, not as the definition of
the platform. A second programme must be able to use the same services without
forking the codebase or inheriting MPOTA policy.

## Motivation

Many successful activation programmes deliberately limit their geographic
scope. That leaves operators who live near ordinary local or municipal parks
with fewer practical opportunities, even when a legitimate public place is a
short walk or short drive away. MyOTA is motivated by the hypothesis that
local geography can make amateur radio more accessible, especially for:

- short spontaneous activations;
- urban and suburban operators;
- QRP, 2 m/70 cm, satellite, and other modes that benefit from nearby sites;
- operators who cannot routinely travel long distances; and
- communities that want to maintain knowledge of places near them.

The project should validate that hypothesis with real parks, real operators,
real QSOs, and reproducible geographic analysis. It should complement
existing programmes rather than attack or replace them. Participation in
another programme neither qualifies nor disqualifies an entity: each MyOTA
programme defines its own policy.

## Product positioning

The public explanation should lead with the user outcome, not the
microservices:

> Local places. Amateur radio. Open infrastructure.

The first experience should make it possible to find a nearby entity, inspect
its verification state, and understand how to participate. The platform story
then explains why the same open API and data model can support many programmes
and integrations.

The initial regional laboratory is Sevilla/Andalucía. The local dataset is a
development and beta asset, not a worldwide completeness claim. Candidate,
approved, and rejected entities must remain visibly distinct; community
proposals are represented as a candidate source, not a separate status.

## Charter principles

1. **Programme independence.** Every programme owns its charter, eligible
   categories, geometry policy, activation validity, minimum QSOs, awards,
   jurisdictions, content, and public terms.
2. **No inherited rules.** MyOTA must not copy or silently impose POTA, MPOTA,
   WWFF, or another programme’s rules, thresholds, exclusions, or award
   definitions.
3. **Open, API-first infrastructure.** Capabilities are exposed through
   versioned contracts and explicit events so public web, mobile clients, GIS
   tools, and logging applications can interoperate.
4. **Evidence before approval.** Imported data is candidate data. Community
   proposals and scoped approver decisions are audited, source-aware, and
   reversible until an approved entity is retired under the platform’s safety
   rules.
5. **Local stewardship without unilateral authority.** Operators may improve
   nearby-entity metadata and evidence, but programme-scoped approvers remain
   responsible for authoritative lifecycle decisions.
6. **Privacy and identity by design.** Amateur-radio operators may have one
   primary callsign, additional callsigns, or SWL participation. Public
   callsign display and activity visibility are programme-configurable.
7. **Reproducibility.** Rule versions, source provenance, aggregation jobs,
   imports, corrections, and award issuances must be explainable later.
8. **Interoperability.** MyOTA should coexist with existing programmes and
   return external references where useful; it should not require users to
   abandon other tools or programmes.

## Community and launch intent

The project is intended to grow through visible utility and participation:

- publish an Explorer-style map before making broad launch claims;
- use Sevilla as a controlled beta with local operators and verified parks;
- invite GIS, programme, club, and logging-tool contributors to critique the
  design;
- publish reproducible datasets and methodology where licensing permits; and
- measure distinct parks activated, successful activations, QSOs, repeat
  participants, geographic accessibility, and external contributions rather
  than GitHub stars alone.

These are product and community goals, not programme eligibility rules. They
belong in the public roadmap and operating plan, not in default programme
configuration.

## Scope boundary

The platform owns reusable primitives and safe execution. Programme owners
own the policy decisions consumed by those primitives. The current ownership
map is maintained in the repository-map document, while the remaining
programme-configuration work is tracked in
the programme-configuration-gap-analysis document.
