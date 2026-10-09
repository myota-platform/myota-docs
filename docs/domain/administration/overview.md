# Administration web UX

The administration web is a control-plane application. It should make the
current scope and the next safe action obvious without requiring an
administrator to understand service boundaries or database terminology.

## Workspace structure

The shell groups work into domain workspaces and platform health:

- **Overview**: dashboard and service signals.
- **Programme setup**: programme configuration, policy, content, and the
  shared entity-category catalogue.
- **Geodata**: review queue, programme-independent imports, entity management,
  and the read-only map explorer.
- **Operations & access**: activations/QSOs, award certificates, and users and
  roles.
- **Platform health**: Grafana observability and authenticated
  [NATS / JetStream status and history](../../operations/messaging/jetstream-admin-status.md).

The programme scope selector is in the top bar so it remains visible while an
administrator moves between programme-owned pages. “All programmes” is an
intentional scope; platform-wide geodata imports and unassigned entities must
not be hidden by a programme filter.

## Import workspace

The Geodata Imports page uses an in-page detail workspace instead of a large
blocking summary dialog:

1. Intake stays at the top and accepts pasted or uploaded source data.
2. The left column contains the preprocessing queue, including uploaded/queued,
   processing and ready-for-validation runs. It includes older active runs,
   not just the first history page.
   Pending uploads and queued/active preprocessing runs have a **Cancel** action
   with a confirmation. Active work reports `CANCELLING` until the worker stops
   at a safe checkpoint; completed preprocessing cannot be cancelled.
3. Selecting a run fills the detail panel on the right and keeps the queue
   visible for comparison.
4. Validation, duplicate comparison, promotion, refresh, and finalization are
   actions within that selected-run context.
5. The duplicate/location map remains a focused modal because the comparison
   needs temporary map space and does not replace the selected import context.
6. Complete import history is below the workspace, paged ten runs at a time.

File uploads use owner-scoped resumable sessions. Each new attempt gets a fresh
idempotency key, retained through pauses, transient errors and lost responses;
submitting the same completed file later creates a new import. Completed parts
are checksum-checked against the reselected original file. Transfer, server
verification and preprocessing are separate progress stages. Pause/resume and
discard controls preserve server progress; a file still undergoing verification
must be resumed rather than aborted. File input state is reset after success.

History and the selected detail refresh every five seconds while the page is
visible; the complete active queue is reconciled every thirty seconds. Summary
counts use authoritative source totals, staging counts and worker statistics.
Selections survive candidate pagination, including select-all across pages.
Actions are disabled while submitting, and records are not offered for promotion
before preprocessing finishes. Finalization retains the summary, not staged
records or obsolete action controls.

Entity edits submit the database revision with `If-Match`. A concurrent edit
returns a conflict instead of overwriting another administrator's change. The
error offers **Reload selected entity**, with confirmation before discarding
unsaved edits. Successful geometry/category/name/location saves refresh the
selected revision. Bulk approval only applies to CANDIDATE entities.

This layout makes a long-running upload observable and prevents the user from
losing the history list behind a modal. On narrow screens the columns stack and
the detail panel follows the queue.

## Geodata review and entity management

Review and management are separate workflows:

- **Review queue** is for filters, paged selection, map centering, lifecycle
  decisions, and bulk approval or global-admin deletion.
- **Entity catalogue** is for name and location metadata, category assignment,
  geometry editing, GIS administration, audit history, and permanent deletion.

Selecting a result centers the Leaflet map. Entity Catalogue now opens a
tabbed editor immediately, without scrolling to sections below the map. The
catalogue map is optional/collapsible; geometry has its own map workspace in
the editor. Tabs separate name/categories, location, geometry, source comparison
and audit/deletion, with previous/next navigation inside the current result
page. Filters, pagination, page position and batch selection remain in place.
Unsaved drafts require confirmation before closing, changing entities or
leaving the route. Saving one section preserves other drafts and refreshes
the database revision, without reselecting/scrolling the entity. Review
decisions remain inline on the review page. See the [catalogue editor
guide](entity-catalogue-editor.md) and [navigation diagram](../../architecture/diagrams/entity-catalogue-editor.md).
Geometry editing is opt-in; the map remains read-only until **Edit geometry
vertices** or a replacement drawing action is selected. Destructive actions
retain the impact warning and are only rendered for authorized roles.
Checkbox selection for bulk actions is independent from the focused entity:
opening an entity for inspection does not replace the selected batch. Bulk
deletion confirms the selected jobs together, then reads their durable job
resources until they complete or fail. The UI reports completed deletion only
after the server confirms completion; queued or failed jobs stay visible for
status checks or retry.

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
