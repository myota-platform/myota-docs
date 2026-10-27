# Awards and programme execution

## Ownership and API boundary

Programme configuration remains owned by `myota-programme-service`, including
the programme's own policy and theme. Award definitions and execution belong to
`myota-activity-service` because they evaluate activation/QSO facts and create
permanent issuance records. The two concerns are related by `programmeSlug`
and versioned events; the activity service does not copy or inherit rules from
another programme.

Activity and awards are exposed by the same bounded service and port. The local
runtime uses port `8004` for both `/v1/activations` and `/v1/awards`; the gateway
routes both path families to the same deployment.

## Award lifecycle

An award is linked to one programme and has a programme-owned code, category,
condition tree, achievement metric, configurable levels, print profile,
background asset and certificate template. Categories are `HUNTER` and
`ACTIVATOR`. A definition follows this lifecycle:

```text
DRAFT -> UNDER_REVIEW -> APPROVED -> PUBLISHED
                 \-> CHANGES_REQUESTED -> DRAFT
```

Publication requires an explicit effective date. Published definitions are
immutable from the administration workflow; a later change is a new draft.

## Conditions and levels

The condition is a programme-owned recursive JSON AST. `AND`, `OR`, `ALL`,
`ANY`, and `NOT` combine leaf conditions. The initial evaluator supports
`QSO_COUNT`, `ACTIVATION_COUNT`, `UNIQUE_CALLSIGNS`, `UNIQUE_ENTITIES`,
`ENTITY_TYPE`, and programme-selected `FIELD` facts with `GTE`, `GT`, `EQ`,
`LTE`, and `LT` comparisons.

Levels are independently configured by administrators, for example 10, 50,
and 100. An evaluation returns progress and eligible levels. A participant can
request an eligible level, and reaching a later level creates a separate
request while preserving earlier issuance history.

For participant requests, the activity service calculates progress from its own
activation/QSO records. Activator progress is derived from activations owned by
the operator; hunter progress uses QSO records carrying the participant's
`hunterId`. Trusted service/admin callers may still submit an explicit facts
snapshot for reconciliation and tests. Participant requests use the account's
`identity.me` scope and cannot request for another account.

QSO ingestion preserves normalized worked callsign, worked-station identity,
timestamp, band, mode, RST, source, import identity and optional
`workedEntityId`. The service validates supported band/mode values, activation
time windows and deduplication keys. Programme rules may further restrict
bands, modes, location, duration and minimum QSOs; those checks are evaluated
when an activation closes. Corrections are proposed and reviewed with an
explicit audit status.

Activation start captures the programme rule snapshot, operator/callsign
context and location. Close evaluates validity, location/rule violations and
produces `CLOSED` or `CLOSED_INVALID` without deleting the historical QSO
facts. Verified callsign authorization remains programme-aware and does not
make a platform-wide eligibility assumption.

The service-owned relational schema uses indexed QSO rows and aggregate
membership tables for activators and hunters. Award progress is materialized by
versioned background jobs, so participant requests do not rescan all QSOs.
Award definitions can be explicitly retired; existing progress and issuance
records still point to the original definition/version.

ADIF is uploaded to SeaweedFS through its S3-compatible API only after size and malware gates, then parsed and
validated asynchronously. Valid rows enter the same COPY/deduplication path as
manual batches. Failed records remain visible in the import result and trigger
notifications; the original upload is retained for reconciliation.

## Certificate assets and print readiness

Backgrounds and manager signatures are registered as image metadata pointing to
an object key in SeaweedFS or another S3-compatible store. The service selects
their bucket by asset kind; clients cannot override it. Editable award
backgrounds use `myota-award-assets`, manager signatures use
`myota-award-signatures`, and generated issued PDF certificates use
`myota-certificates`. The service records the bucket, endpoint, media type,
dimensions and optional checksum. The admin web
uploads PNG/JPEG bytes through authenticated `PUT /v1/awards/assets/{assetId}/content`
(20 MiB / 16 million pixel bounds). Named backgrounds and signatures populate
the designer selectors. Content replacement uses authenticated binary upload.
See the [programme and award designer guide](admin-designer.md).
The certificate template stores normalized `x`, `y`, `width`, and `height`
positions for the award name, optional Award Text, callsign, participant name,
date obtained, manager name and manager signature. Award Text is a separate
optional value (up to 200 characters); older definitions without it remain
compatible and render no placeholder. Its default element is below the award
name at `y = 0.21`. The admin web provides a draggable preview plus numeric
adjustments. The six original default elements are restored for empty legacy
templates, and custom text can be added. A transient mock-data PDF preview opens
separately without saving or issuing an award, using the same orientation-aware
print renderer.

Published definitions are immutable. Editing one creates a new draft revision
with the same programme and award code, an incremented version, and a link to
the source definition. The API requires the version loaded by the editor and
rejects stale edits with a conflict. The previous published version and its
issuances remain unchanged; the new draft uses the normal review and publication
flow.

For 300 DPI print output, the recommended profiles are:

| Profile | Portrait | Landscape |
|---|---:|---:|
| A4 | 2481 × 3508 px | 3508 × 2481 px |
| Letter | 2550 × 3300 px | 3300 × 2550 px |

The exact output width and height are `ceil(page millimetres / 25.4 × DPI)`;
supported DPI is 150–300. The renderer applies a centered cover crop and resizes
the source to those dimensions without modifying the original asset. Print
readiness is based on the remaining source detail after the crop: the usable
source area must be at least 150 DPI for the selected page. A source with
different pixel dimensions is accepted when it meets that minimum; a small
source that does not meet it is blocked from publication. The issuance record
permanently stores the award name, Award Text, callsign, participant name, date
obtained, manager name, signature asset reference, background reference, print
profile and element positions.

When the background and signature objects are present and the Pillow renderer is
available, issuing an award renders a PDF into the certificate bucket and the
participant can obtain a short-lived download URL. If assets are not yet
available, the issuance remains permanent with `renderStatus:
WAITING_FOR_ASSETS`; an administrator can retry `/render` later. This keeps
achievement history durable even when object storage or rendering is
temporarily unavailable.

The [Phase 6 award-layout qualification record](../../operations/evidence/phase6-awards-layout-qualification-2026-10-27.md)
lists the code and CI coverage and the limits of the preview performed against
the current `myota-test` release.

The public programme web lists published awards, signs participants in through
the identity service, displays server-calculated progress, and submits eligible
level requests. Issuance remains a programme/admin action so the manager name
and signature are explicit and auditable.

## Events

The activity service emits `awards.definition.saved.v1`,
`awards.definition.published.v1`, `awards.request.created.v1`, and
`awards.issued.v1` through the same durable outbox used for activation and QSO
events. Geodata review and identity security events are consumed into the
activity notification table. Consumers should use event IDs for idempotency;
the notification table also uses deduplication keys, including the reached
award levels, so retries do not duplicate delivery while newly reached levels
can generate a new qualification notice. Treat issuance records as append-only
history.

Public execution endpoints expose masked activation history, leaderboards and
JSON/CSV result downloads. Full callsign display is not the default public
representation.
