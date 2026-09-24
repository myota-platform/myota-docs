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

## Certificate assets and print readiness

Backgrounds and manager signatures are registered as image metadata pointing to
an object key in MinIO or another S3-compatible store. The service records the
bucket, endpoint, media type, dimensions and optional checksum. The admin web
uploads through a short-lived presigned URL, with a small JSON/base64 fallback
for local administration. The certificate
template stores normalized `x`, `y`, `width`, and `height` positions for the
award name, callsign, participant name, date obtained, manager name and manager
signature. The admin web provides a draggable preview plus numeric adjustments.

For 300 DPI print output, the recommended profiles are:

| Profile | Portrait | Landscape |
|---|---:|---:|
| A4 | 2481 × 3508 px | 3508 × 2481 px |
| Letter | 2550 × 3300 px | 3300 × 2550 px |

The service rejects publication when the background aspect ratio is outside the
configured tolerance or the image is materially below the recommended
resolution. The issuance record permanently stores the award name, callsign,
participant name, date obtained, manager name, signature asset reference,
background reference, print profile and element positions.

When the background and signature objects are present and the Pillow renderer is
available, issuing an award renders a PDF into the certificate bucket and the
participant can obtain a short-lived download URL. If assets are not yet
available, the issuance remains permanent with `renderStatus:
WAITING_FOR_ASSETS`; an administrator can retry `/render` later. This keeps
achievement history durable even when object storage or rendering is
temporarily unavailable.

The public programme web lists published awards, signs participants in through
the identity service, displays server-calculated progress, and submits eligible
level requests. Issuance remains a programme/admin action so the manager name
and signature are explicit and auditable.

## Events

The activity service emits `awards.definition.saved.v1`,
`awards.definition.published.v1`, `awards.request.created.v1`, and
`awards.issued.v1` through the same durable outbox used for activation and QSO
events. Consumers should use event IDs for idempotency and treat issuance
records as append-only history.
