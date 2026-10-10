# Admin workspaces and navigation

Implemented 9 October 2026. This is an API-only Vue client reorganization, not a
change to service ownership, programme rules or backend authorization. Existing
URLs remain valid. The green/white MyOTA visual style is retained.

## Review and decisions

| Finding | Implemented response |
| --- | --- |
| Every sidebar page appeared regardless of permissions; failures followed navigation. | Shared workspace definitions drive navigation, headings and route guards from the canonical identity `/me` response. Unauthorized deep links explain access without signing out. |
| Shared Master data appeared under programme setup. | **Entities → Entity categories**; programme assignments stay in Programmes. |
| Rules, policy and certificate names overlapped. | **Rules & policies** for programme-owned schemas; **Awards & certificates** for conditions, levels, artwork and issuance. |
| Header and page programme selectors repeated and disagreed. | One selector in programme-owned workspaces; read-only header context. Shared entity/import pages remain Platform-wide. |
| Users, roles and security were a long stacked page. | Three deep-linkable tabs in **Users & access**. Built-in roles appear immutable. |
| User saves could flatten existing role restrictions. | Omit unchanged grants; retain every programme/jurisdiction/category assignment when role selection changes. New grants are explicitly described as unscoped. |
| Published records looked editable; errors appeared as successes. | Read-only styling, disabled mutations, failure alerts and separate success feedback. Content effective dates are chosen after approval and persisted at publication. |
| A forbidden/unavailable overview request broke all dashboard data. | Permission-specific requests and independent failure handling; unavailable counts show “—”, not invented zeros. |
| Refresh and spacing varied between pages. | Shared page headings, refresh/action areas and feedback; consistent labels, spacing, responsive grids and keyboard focus. |
| Map exploration silently stopped after its first API page. | Loaded/total counts and bounded **Load next 100**, not unbounded automatic fetching. |

## Page map

| Group | Workspace and stable URL |
| --- | --- |
| Start | Overview `/dashboard`: available tasks and live signals |
| Entities | Entity catalogue `/entity-management`; Entity map `/entity-map` |
| Entities | Geodata review `/geodata`; Geodata imports `/geodata-imports` |
| Entities | Entity categories `/master-data`: shared definitions |
| Programmes | Programmes `/programmes`: initiative configuration and memberships |
| Programmes | Rules & policies `/policies`; Content & translations `/content`; Awards & certificates `/awards` |
| People | Users & access `/identity`; tabs `?tab=users`, `roles`, `security` |
| Activity | Activations & QSOs `/activity` |
| Platform health | Current NATS / JetStream `/jetstream` (planned for retirement after [Surveyor cutover](../../observability/nats-surveyor-migration.md)); SeaweedFS storage `/object-storage`; Metrics & dashboards `/observability/` |

Use **Find a workspace** to filter authorized navigation. Overview's **Start a
task** offers direct workflow entry points. On narrow screens the menu button
opens navigation, keyboard focus enters it, Tab stays inside, Escape restores
the toggle and selecting a workspace closes it. A skip link leads to the main
workspace. All operational dates/times remain UTC.

## Access and read-only fields

- `/v1/identity/me` supplies resolved roles/scopes; login's basic account alone
  is not sufficient. Bootstrap is deduplicated and fenced against session
  changes. Programme-choice outages do not destroy a verified session.
- Wildcard and existing `GLOBAL_OPERATOR`/`GLOBAL_ADMIN` compatibility access
  is retained. Service APIs still enforce every action and entity/programme
  restriction. Hiding controls is usability, not a security boundary.
- Shared categories are readable by geodata staff; changes require
  `programme.admin`, matching the owning API.
- User edits require `identity.admin`; role assignment additionally requires
  `identity.roles.assign`; role-definition edits require `identity.roles.manage`.
  Non-global staff cannot modify global-account grants. Built-in roles remain
  immutable. Existing scoped assignments are displayed and preserved.
- Awards use `awards.read`/`awards.admin`, not an inferred `activity.admin` grant.
  Readers cannot change definitions, upload artwork or request admin previews.
  Extending the permission catalogue is separate identity-policy work.
- Name/category/geometry edits require review access. Geometry-type conversion
  and location administration retain dedicated GIS permissions. Permanent
  deletion stays global-only with durable jobs and impact warnings.
- Stable role/category codes, provider location codes, derived Maidenhead
  values and published version contents are read-only. Draft edits and
  lifecycle/publication controls are distinct.

## Preserved workflows

The catalogue keeps focused tabs, draft guards, revisions, previous/next entity
navigation and independent batch selection. Leaflet/Geoman drawing, clustering,
OSM attribution, retired restrictions, source/audit comparison, single/bulk
deletion warnings and job polling remain. Imports retain resumable transfers,
preprocessing, duplicate maps, cross-page selection, promotion, cancellation,
finalization and paged history. Award design retains default fields, dragging,
print orientation, named PNG/JPEG assets and mock-data preview PDFs. Programme
editing retains custom rules/themes.

No new API endpoint, direct database access, contract migration, Compose port
change or Helm architecture change was needed. The admin image rolls out through
the existing GHCR digest → Fleet → Helm path; local Compose uses that same image.

See [diagram](../../architecture/diagrams/admin-workspaces.md),
[catalogue guide](entity-catalogue-editor.md),
[designer guide](../awards/admin-designer.md) and
[delivery evidence](evidence/admin-workspaces-2026-10-09.md).
