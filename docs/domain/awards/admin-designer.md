# Programme editing and award certificate design

## Programme editor

Selecting a programme loads its detail resource through `GET /v1/programmes/{slug}`.
Programme Identifier is the permanent URL/API slug: it is editable when creating
a programme and read-only afterwards. Programme Name, description, policy,
theme and category assignments remain editable. Identifier and name inputs are
top-aligned even when only one has explanatory text.

The editor snapshots API data before attaching it to Vue reactive state. It
preserves unknown programme-owned rules, nested minimum-QSO fields and theme
metadata on save rather than silently dropping them. A late selection response
cannot replace a newer selection. Category membership still uses the explicit
relationship APIs; the browser never accesses a database directly.

## Award designer

Choose a programme and an existing draft, or create a new award. List summaries
are not used as editable records: the UI fetches `GET /v1/awards/{awardId}`.
Published definitions remain read-only. Draft edits use `PATCH /v1/awards/{awardId}`.
To edit a published award, select **Create editable version**. Saving creates a
new draft version linked to the published definition; it does not alter the
published version or existing issuances. The request includes the version that
was loaded. If another edit wins first, the API returns a conflict and the
administrator must reload before trying again. The new draft keeps the same
programme and award code and can proceed through the normal review and
publication workflow.
Existing template coordinates/styles, print metadata and legacy background keys
are retained when editing. Effective dates are displayed and entered in UTC; an
unchanged date retains its original instant (including seconds) rather than
shifting on a draft save. Empty/missing older templates receive the six defaults:
award name, callsign, participant name, date obtained, manager name and signature.
See the platform-wide [UTC time policy](../../platform/utc-time-policy.md).
The defaults also appear immediately when opening the page for the first time.

The separate **Award Text** field holds up to 200 characters; it does not replace
the award name or description. Existing awards with no value behave as though it
were blank. The default layout places its text element immediately below the
award name at normalized `y = 0.21`. Fields can be dragged or positioned using
normalized coordinates. Restore default elements asks before replacing a layout;
additional custom text is supported.

A4/Letter and portrait/landscape control canvas proportions and PDF dimensions.
At a chosen DPI, the pixel width and height are each rounded up from the page's
millimetre dimensions. Supported print settings are 150–300 DPI. Changing page
orientation updates the displayed target dimensions.

### Backgrounds and signatures

The artwork section accepts PNG and JPG/JPEG. Enter a display name, choose
Background or Signature, and select the file. The browser derives media type and
dimensions; it generates a unique object key. Registered backgrounds populate
the Background object key selector by display name; registered signatures have
their own selector. A legacy key is retained as an explicit option, not erased.
The selected images appear in the designer once their content is available.

Uploads register metadata, then send raw image bytes to the activity API. Both
browser and service enforce 20 MiB and 16 million pixels; the service verifies
actual image format/dimensions and applies its configured malware-scanning gate.
The service selects buckets, ignoring caller-supplied storage destinations:

| Purpose | Bucket |
| --- | --- |
| Editable backgrounds | `myota-award-assets` |
| Manager signatures | `myota-award-signatures` |
| Permanently issued PDFs | `myota-certificates` |

No browser-reachable S3 endpoint is required. The UI uploads image bytes through
the authenticated content resource; the service selects and validates the
storage destination.
An interrupted upload can leave registered metadata without content; registration
alone does not imply print readiness. No existing asset or award is deleted.
The renderer resizes a supported source background to the exact selected page
dimensions using a centered cover crop; it does not replace or mutate the
registered source. Print readiness checks the usable crop area, which must retain
at least 150 DPI of source detail. A source can therefore have different pixel
dimensions from the output and still be usable. If it is too small after
cropping, publication is blocked with a message to choose a higher-resolution
image. For reference, A4 portrait at 300 DPI is 2481×3508 pixels and Letter
portrait is 2550×3300 (landscape swaps the dimensions).

### Preview PDF

Generate preview PDF opens a separate window and renders the current **unsaved**
layout with mock `EA7TEST`, Demo Radio Operator and the current date. The selected
background/signature and manager name are included. No award request, issuance,
outbox event or PDF object is created. A blank-paper preview is allowed; an
unregistered or missing selected asset reports an error. Allow popups for the
admin origin if the browser blocks the window.

The renderer is shared with issuance, but preview is bounded: 64 KiB design
request, at most 30 elements, 150–300 DPI, and one concurrent preview per API pod.
A busy pod returns 429 rather than allocating unbounded raster memory. Issued
certificate rendering still uses its existing background-worker/resource flow;
preview is a transient, synchronous design operation, not that durable job.

## API and repository ownership

| Method/resource | Purpose | Authorization |
| --- | --- | --- |
| `POST /v1/awards/assets` | Register named image metadata | `awards.admin` |
| `PUT /v1/awards/assets/{assetId}/content` | Replace image bytes; verify and store | `awards.admin` |
| `GET /v1/awards/assets/{assetId}/content` | Return bounded image content/metadata | `awards.read` or `awards.admin` |
| `POST /v1/awards/previews` | Return transient PDF bytes as base64 JSON | `awards.admin` |

The canonical OpenAPI and dependency-light clients live in `myota-contracts`.
The handler and shared renderer live in `myota-activity-service`; integration
copies in platform/deploy remain synchronized with that owner. The new content
resource is preferred over the old JSON/base64 write action. All paths use the
existing activity port 8004 and gateway prefix; Compose/Helm topology, secrets
and bucket configuration do not change. Fleet rolls the images using recorded
digests, and Helm rendering remains in GitHub Actions.

## Verification

Browser regressions in `tests/browser/programmeAwards.spec.ts` cover existing
programme/detail loading, alignment, metadata-preserving saves, default fields,
legacy award loading, named PNG/JPG uploads, selection and separate-window
previews. They use isolated API fixtures, not live data. Activity HTTP tests in
`tests/test_award_design.py` cover binary uploads over the old 700 KB limit,
validation/authorization, draft round trips, preview backpressure, all four
paper/orientation combinations and absence of persisted preview side effects.
See [delivery evidence and remaining verification limits](evidence/programme-awards-2026-10-09.md).
Published award revision, Award Text and background-resizing qualification is
recorded in the [Phase 6 award-layout evidence](../../operations/evidence/phase6-awards-layout-qualification-2026-10-27.md).
