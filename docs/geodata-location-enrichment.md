# Geodata location enrichment

`myota-geodata-service` enriches location metadata from an entity's
representative point using BigDataCloud's server-side Reverse Geocoding to City
API. Enrichment is asynchronous: the entity mutation and request event commit
together, and a durable geodata worker performs the provider lookup.

## When enrichment runs

The entity lifecycle is the source of truth for timing:

1. **Import preprocessing** validates and normalizes source features but does
   not call the reverse-geocoding provider. Large imports therefore do not wait
   on provider latency, and a provider failure cannot reject a valid feature.
2. **Entity materialization** queues an `only missing` lookup when a newly
   created candidate/approved entity lacks required location metadata. An
   imported update queues a full refresh only when its geometry changed.
3. **Geometry edits and geometry-type changes** queue a refresh after the new
   normalized geometry and centroid are saved. Automatic values are refreshed
   for the new location; explicitly manual fields and their corresponding
   codes remain authoritative.
4. **Manual metadata edits** do not trigger a lookup. Releasing fields from the
   manual override set clears those values and queues an `only missing` lookup.
5. **Administrator retry** is available in Entity Management's Location
   metadata section while required values are missing. It requests enrichment
   for the current geometry; complete entities cannot be refreshed through
   this button.

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
[`myota-geodata-service` commit `0b3655e`](
https://github.com/myota-platform/myota-geodata-service/commit/0b3655e).

During the 8 October 2026 live diagnosis, the outbox contained five
location-enrichment request events and none were pending publication. The
durable consumer was subscribed, and four distinct referenced entities had
both geometry and a stored database centroid. The requests were consumed but
failed with `SKIPPED_NO_CENTROID` because the application projection omitted
that centroid. The code fix restores it. Existing failed events have already
been acknowledged, so retry them from Entity Management after the fix is
deployed; they will not be replayed automatically.

The geodata pod currently sources `BIGDATACLOUD_API_KEY` from the optional
`myota-geodata-enrichment` Secret's `api-key` field. That named Secret was not
present in the namespace during diagnosis. The operator clarified that the
credential is stored in an existing Kubernetes Secret under `password`, with
`geo-database-url` also present. The current Helm wiring does not map that
source into the geodata pod, so the issue is a Secret-reference mismatch—not
proof that the credential is absent from Kubernetes. The deployment guide
currently defines `myota-postgres/password` as the PostgreSQL superuser
password. Confirm that this exact value is intentionally also the BigDataCloud
key before mapping it into `BIGDATACLOUD_API_KEY`; never send a database
password to the provider by assumption. Never put secret values in this
document, logs, or a commit.

The centroid projection fix was deployed to K3s as image digest
`sha256:4547e6f39091b7027c42758ad3e164e582f287f133faf3bf0a5758b18cc56775`.
Both the geodata API and processing worker rolled successfully, and the public
gateway health check returned `ok`. The already-failed enrichment requests
were acknowledged before the fix, so retry them from Entity Management after
the provider Secret reference is confirmed and wired.

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

## Configuration

For local development, put the server-side key in the ignored
`myota-geodata-service/.env` file:

```dotenv
BIGDATACLOUD_API_KEY=...
BIGDATACLOUD_LOCALITY_LANGUAGE=en
BIGDATACLOUD_TIMEOUT_SECONDS=10
```

Production Kubernetes deployments provide the key through the configured
location-enrichment Secret (default `myota-geodata-enrichment`, key `api-key`)
to both the geodata API and its worker. The client-side free
endpoint is not used: its fair-use terms prohibit server-side and batch
lookups of stored or imported coordinates. The service caches rounded
centroids in-process and treats provider failures as non-fatal to entity or
import persistence. Provider status values include `ENRICHED`, `SOURCE_DATA`,
`FAILED`, `NOT_CONFIGURED`, and `SKIPPED_NO_CENTROID`.
