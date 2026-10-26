# Geodata location enrichment

The Geodata worker enriches location metadata from an entity's representative
point using BigDataCloud's server-side Reverse Geocoding to City API.
Enrichment is asynchronous: the entity mutation and request event commit
together, and a durable worker performs the provider lookup.

> **MyOTA Next chart configuration:** use the current monorepo settings below.
> The dated deployment notes later in this page include legacy split-repository
> evidence and should not be used as the current chart configuration.

## MyOTA Next API configuration and Secret

The `geodata-worker` is the only MyOTA Next component that calls the provider.
The Geodata API and Admin browser do not need the provider credential. The
chart reads the API key from the Kubernetes Secret configured by
`secrets.geodataEnrichment` (default `myota-geodata-enrichment`), key
`api-key`. This Secret reference is optional: with no key, the worker remains
available and records reverse geocoding as `NOT_CONFIGURED`.

The non-secret Helm settings are:

```yaml
geodataWorker:
  locationEnrichment:
    reverseGeocodeUrl: https://api-bdc.net/data/reverse-geocode
    localityLanguage: en
    timeoutSeconds: '10'
```

`reverseGeocodeUrl` defaults to the server-side BigDataCloud
`/data/reverse-geocode` endpoint; `localityLanguage` defaults to English and
accepts ISO 639-1 language codes; `timeoutSeconds` defaults to 10. The worker
rounds representative coordinates to five decimal places and sends the API key
as the provider's `key` query parameter. Do not log or trace the full provider
request URL. See BigDataCloud's [Reverse Geocoding to City API
reference](https://www.bigdatacloud.com/geocoding-apis/reverse-geocode-to-city-api).

Use the server-side API for stored entity coordinates. BigDataCloud's free
client-side endpoint is limited to a user's current location obtained on that
client; it must not be used for stored or batch coordinates. Review the
provider's [fair-use guidance](https://www.bigdatacloud.com/docs/article/why-is-reverse-geocoding-api-free)
and select an account plan/quota appropriate for the expected backfill volume.

Provision the key from a permission-restricted file outside the repository or
through the approved secret manager. For example:

```sh
kubectl -n myota-test create secret generic myota-geodata-enrichment \
  --from-file=api-key=/secure/path/bigdatacloud-api-key \
  --dry-run=client -o yaml | kubectl apply -f -
```

Do not use `--from-literal`, Helm `--set`, a checked-in values file, or a
ConfigMap for the key. Use separate provider credentials and quota budgets for
test and production. After creating or rotating the Secret, restart the
`geodata-worker` Deployment so the process reads the new environment value.
The API key is never sent to the browser. Keep asynchronous imports and
backfills within the provider plan's monthly request allowance; failed lookups
remain retryable and do not reject entity creation or promotion.

## When enrichment runs

The entity lifecycle is the source of truth for timing:

1. **Import preprocessing** validates and normalizes source features but does
   not call the reverse-geocoding provider. Large imports therefore do not wait
   on provider latency, and a provider failure cannot reject a valid feature.
2. **Entity materialization** queues an `only missing` lookup when a newly
   created candidate/approved entity lacks required location metadata. An
   imported update queues a full refresh only when its geometry changed.
3. **Lifecycle promotion** to `CANDIDATE` or `APPROVED` queues an `only
   missing` lookup when required provider-managed fields are absent. The
   status change and enrichment request are written in the same database
   transaction. Provider calls remain asynchronous, so provider failure does
   not reject a valid promotion.
4. **Geometry edits and geometry-type changes** queue a refresh after the new
   normalized geometry and centroid are saved. Automatic values are refreshed
   for the new location; explicitly manual fields and their corresponding
   codes remain authoritative.
5. **Manual metadata edits** do not trigger a lookup. Releasing fields from the
   manual override set clears those values and queues an `only missing` lookup.
6. **Administrator retry** is available in Entity Management's Location
   metadata section while required values are missing. It requests enrichment
   for the current geometry; complete entities cannot be refreshed through
   this button.

While a selected entity is `QUEUED`, Entity Management refreshes its resource
every two seconds so asynchronously persisted values appear without a manual
page reload. Polling stops when the request reaches `COMPLETED` or `FAILED`, the
administrator selects another entity, or the page is closed.

Each request includes the entity ID, a request ID, and a hash of the geometry
used. The worker checks the request/hash before and after the remote call. A
late result from an old geometry is discarded. Redelivery after a successful
database commit is idempotent and does not repeat the external lookup. Provider
failure leaves existing values untouched and records `FAILED`, so an
administrator can retry if metadata is still missing.

The preferred retry resource is
`POST /v1/geodata/entities/{entityId}/location-enrichment-requests`. It
requires GIS location-management permission and returns `202` after the request
is committed to the transactional outbox. The durable pull consumer is
`geodata-location-enrichment-v1` on
`myota.geodata.entity.location-enrichment.v1`. See the [event contract](https://github.com/myota-platform/myota-contracts/blob/main/contracts/events.md)
for message and acknowledgement semantics.

## Durable coordinates and troubleshooting

The worker reloads each entity from the relational row repository before
calling the provider. That projection must include a `{lon, lat}` centroid,
reconstructed from the dedicated PostGIS `centroid` column and falling back to
`ST_Centroid(geom)` for older rows. Without it, an event can be published and
consumed normally while the lookup ends as `SKIPPED_NO_CENTROID`; this is not a
JetStream delivery failure. The regression fix is in
[`myota-geodata-service` commit `0b3655e`](https://github.com/myota-platform/myota-geodata-service/commit/0b3655e).

During the 8 October 2026 live diagnosis, the outbox contained five
location-enrichment request events and none were pending publication. The
durable consumer was subscribed, and four distinct referenced entities had
both geometry and a stored database centroid. The requests were consumed but
failed with `SKIPPED_NO_CENTROID` because the application projection omitted
that centroid. The code fix restores it. Existing failed events have already
been acknowledged, so retry them from Entity Management after the fix is
deployed; they will not be replayed automatically.

The configured provider reference is the dedicated Kubernetes Secret
`myota-geodata-enrichment`, key `api-key`. After the operator provisioned that
Secret, the API and worker pods still had an empty `BIGDATACLOUD_API_KEY`:
Secret-backed environment variables are read when a pod starts, not refreshed
inside an already-running process. Both deployments were restarted. Verification
confirmed the key is now present in each runtime without displaying its value,
and a sanitized live lookup from the geodata pod returned `ENRICHED` for a
Sevilla-area coordinate. The worker log also confirmed its durable
`geodata-location-enrichment-v1` consumer subscribed to the expected subject.
The existing failed events were acknowledged before this recovery and did not
replay automatically. After the provider Secret and worker were active, five
fresh requests were submitted. Live database verification confirmed all five
outbox events were published and processed by the durable consumer; the
corresponding entities reached `COMPLETED` / `ENRICHED`, with country and city
persisted and no enrichment error. This verifies the full entity → outbox →
JetStream → provider → persisted-entity path. Use **Update missing location
data** in Entity Management for any other entities that still lack metadata.

The Admin UI originally refreshed the entity only immediately after queueing,
so it could continue to display old fields after the worker had saved the
result. It now polls the selected entity every two seconds while enrichment is
queued and stops after completion or failure. The fix is in
[myota-admin-web commit `73019a7`](https://github.com/myota-platform/myota-admin-web/commit/73019a7)
and is deployed to K3s.

The centroid projection fix was deployed to K3s as image digest
`sha256:4547e6f39091b7027c42758ad3e164e582f287f133faf3bf0a5758b18cc56775`.
Both the geodata API and processing worker rolled successfully. The Helm chart
now includes `geodataPipeline.locationEnrichment.rolloutRevision` on both pod
templates; increment it in the Spainip values whenever the external Secret is
created or rotated, so Fleet applies the new environment value through Helm.
The Secret itself remains external and is never stored in chart values or Git.

## Entity fields and provenance

The entity exposes:

- `continent` / `continentCode`
- `country` / `countryCode`
- `region` / `regionCode`
- optional `province` / `provinceCode`
- optional `county` / `countyCode`
- `city` (or municipality) and optional `locality`

`regionCode` is the provider's `principalSubdivisionCode`: the first
administrative subdivision after the country. The complete provider response is
retained under `provenance.reverseGeocoding` for auditability. Missing
administrative levels remain null; the service does not infer a county from a
city or duplicate a province as a county. `locationEnrichmentStatus` reports
`QUEUED`, `COMPLETED`, or `FAILED` independently of the provider's
`geocodeStatus`.

The location fields were introduced by
`myota-geodata-service/migrations/004_location_enrichment.sql`. Enrichment
request and status metadata are stored in the entity's service-owned JSON
properties; the asynchronous lifecycle adds no shared infrastructure table.

## Historical split-repository configuration

For local development, put the server-side key in the ignored
`myota-geodata-service/.env` file:

```dotenv
BIGDATACLOUD_API_KEY=...
BIGDATACLOUD_LOCALITY_LANGUAGE=en
BIGDATACLOUD_TIMEOUT_SECONDS=10
```

Production Kubernetes deployments provide the key through the configured
location-enrichment Secret (default `myota-geodata-enrichment`, key `api-key`)
to both the geodata API and its worker. A pod restart is required after
provisioning or rotating that Secret; on K3s, increment
`geodataPipeline.locationEnrichment.rolloutRevision` in the Spainip Helm values
and let Fleet roll both workloads. The client-side free
endpoint is not used: its fair-use terms prohibit server-side and batch
lookups of stored or imported coordinates. The service caches rounded
centroids in-process and treats provider failures as non-fatal to entity or
import persistence. Provider status values include `ENRICHED`, `SOURCE_DATA`,
`FAILED`, `NOT_CONFIGURED`, and `SKIPPED_NO_CENTROID`.

## 14 October 2026 lifecycle fix and backfill

The review and status-update handlers previously persisted promotions without
creating a location-enrichment request. That left existing entities dependent
on a later geometry edit or manual retry. The Geodata fix queues an
`onlyMissing` request in the same transaction for transitions into `CANDIDATE`
or `APPROVED`; rejected entities are not enriched. The existing worker still
checks the request ID and geometry hash and protects manual fields.

The production inventory contained 10 approved entities and no active entity
with required location fields missing. The isolated `myota-test` namespace had
126 approved entities; 116 had geometry but lacked required location values.
Testing values intentionally had `geodataPipeline.locationEnrichment.enabled`
disabled, so prior attempts recorded `NOT_CONFIGURED`. A bounded one-time run
queued those 116 entities as `onlyMissing` and processed them through the
existing enrichment worker logic at one request per second. All 116 reached
`COMPLETED` / `ENRICHED`, and a follow-up database read found no active entity
missing required fields. The test worker was restored to 1/1 ready. The
production database was not changed, and the production key was passed only
through the one-off process; no key was copied into the test namespace or
written to the repository. Future enrichment in `myota-test` remains disabled
until a test provider key is provisioned and its Helm setting is enabled.

The implementation and focused test evidence are tracked in
[Geodata issue #27](https://github.com/myota-platform/myota-geodata-service/issues/27).
The fix was merged to Geodata `main` as `122d4b4` through release PR
[#29](https://github.com/myota-platform/myota-geodata-service/pull/29), and the
main image was published with digest
`sha256:f83e9d13a966afebb32ba4266a92c6fd045e1ad01fcb94750d29dca429c71c31`.
Production is still on the previous digest because Deploy's scheduled
production digest-sync workflow attempted a direct push to protected `main`;
GitHub rejected it because the required `Main PR source policy` check was
missing. [Deploy issue #60](https://github.com/myota-platform/myota-deploy/issues/60)
tracks the compliant rollout path. The fix is in the release image but is not
active in the deployed production API or worker yet.
