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
