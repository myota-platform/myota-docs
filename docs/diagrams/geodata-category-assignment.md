# Geodata category assignment

This diagram shows the ownership and persistence path for a candidate from
either an adapter/import run or a community proposal. Categories are shared
Master data; programme assignment is a separate eligibility relationship.

```mermaid
flowchart LR
  Admin[Admin web\nNew candidate / import / review]
  Community[Community proposal\nuser-submitted feature]
  Catalogue[Programme service\nshared entity category catalogue]
  Geo[Geodata API\nnormalize and authorize]
  Primary[geodata_entity\nentity_type_code = primary]
  Assignments[(geodata_entity_category\nall assignments + is_primary)]
  Snapshot[(Compatibility JSON\nentityTypes projection)]
  Programme[Programme assignment\nzero, one, or many programmes]
  Activity[Activity service\nactivation and QSO eligibility]

  Admin -->|GET catalogue| Catalogue
  Catalogue -->|codes + labels + geometry kinds| Admin
  Admin -->|entity types list| Geo
  Community -->|candidate source| Geo
  Geo -->|first code| Primary
  Geo -->|replace assignment set| Assignments
  Geo -->|compatibility projection| Snapshot
  Catalogue -->|separate assignment API| Programme
  Programme -->|eligible entity reference| Activity
```

Rules:

- `entityTypes[]` must contain at least one stable uppercase category code.
- The first code is exposed as legacy `entityType` and stored as the one
  `is_primary` assignment.
- The relational assignment table is authoritative for multi-category reads;
  the JSON snapshot is retained for compatibility and export.
- Imports and community proposals are programme-independent and always create
  `CANDIDATE` entities. The candidate source distinguishes `ADAPTER_IMPORT`
  from `COMMUNITY_PROPOSAL`.
- A category can be assigned to multiple programmes, and an entity may have
  multiple categories.

The canonical schema is
`myota-geodata-service/migrations/008_entity_category_assignments.sql`.
Synchronized copies are applied by the platform bootstrap and deployment
migration runners.
