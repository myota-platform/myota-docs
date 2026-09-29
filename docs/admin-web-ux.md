# Administration web UX

The administration web is a control-plane application. It should make the
current scope and the next safe action obvious without requiring an
administrator to understand service boundaries or database terminology.

## Workspace structure

The shell groups work into four areas:

- **Overview**: dashboard and service signals.
- **Programme setup**: programme configuration, policy, content, and the
  shared entity-category catalogue.
- **Geodata**: review queue, programme-independent imports, entity management,
  and the read-only map explorer.
- **Operations & access**: activations/QSOs, award certificates, and users and
  roles.

The programme scope selector is in the top bar so it remains visible while an
administrator moves between programme-owned pages. “All programmes” is an
intentional scope; platform-wide geodata imports and unassigned entities must
not be hidden by a programme filter.

## Import workspace

The Geodata Imports page uses an in-page detail workspace instead of a large
blocking summary dialog:

1. Intake stays at the top and accepts pasted or uploaded source data.
2. The left column contains the preprocessing queue and complete import
   history, including active runs.
3. Selecting a run fills the detail panel on the right and keeps the queue
   visible for comparison.
4. Validation, duplicate comparison, promotion, refresh, and finalization are
   actions within that selected-run context.
5. The duplicate/location map remains a focused modal because the comparison
   needs temporary map space and does not replace the selected import context.

This layout makes a long-running upload observable and prevents the user from
losing the history list behind a modal. On narrow screens the columns stack and
the detail panel follows the queue.

## Geodata review and entity management

Review and management are separate workflows:

- **Review queue** is for filters, paged selection, map centering, lifecycle
  decisions, and bulk approval or global-admin deletion.
- **Entity catalogue** is for name and location metadata, category assignment,
  geometry editing, GIS administration, audit history, and permanent deletion.

Selecting a result always centers the Leaflet map. In Entity Catalogue it also
scrolls to the management editor below the map. Geometry editing is opt-in;
the map remains read-only until **Edit geometry** is selected. Destructive
actions retain the impact warning and are only rendered for authorized roles.

## Interaction rules

- Use labels and short help text for every non-obvious field.
- Show loading, empty, error, and stale/active states in the same surface as
  the data they describe.
- Keep selected state visible in both the list and the map.
- Preserve source provenance; edits create audit entries rather than silently
  rewriting imported evidence.
- Use `aria-current` for the active navigation item, keyboard-visible focus,
  live status messages, and responsive layouts without hiding essential
  actions.

These are presentation rules only. Authorization, lifecycle transitions,
deduplication, persistence, and queue semantics remain owned by the relevant
service APIs.
