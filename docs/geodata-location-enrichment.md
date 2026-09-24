# Geodata location enrichment

\`myota-geodata-service\` enriches every imported, drawn, or geometry-edited
entity from its centroid using BigDataCloud's server-side Reverse Geocoding to
City API.

The entity exposes:

- \`continent\` / \`continentCode\`
- \`country\` / \`countryCode\`
- \`region\` / \`regionCode\`
- optional \`province\` / \`provinceCode\`
- optional \`county\` / \`countyCode\`
- \`city\` (or municipality) and optional \`locality\`

\`regionCode\` is the provider's \`principalSubdivisionCode\`: the first
administrative subdivision after the country. The complete provider response is
retained under \`provenance.reverseGeocoding\` for auditability. Missing
administrative levels remain null; the service does not infer a county from a
city or duplicate a province as a county.

The canonical schema change is
\`myota-geodata-service/migrations/004_location_enrichment.sql\`. The platform
and deployment repositories contain synchronized copies because their migration
runners bootstrap the geodata database.

## Configuration

For local development, put the server-side key in the ignored
\`myota-geodata-service/.env\` file:

\`\`\`dotenv
BIGDATACLOUD_API_KEY=...
BIGDATACLOUD_LOCALITY_LANGUAGE=en
BIGDATACLOUD_TIMEOUT_SECONDS=10
\`\`\`

Production Kubernetes deployments must provide the key through the
\`myota-geodata\` Secret under \`bigdatacloud-api-key\`. The client-side free
endpoint is not used: its fair-use terms prohibit server-side and batch
lookups of stored or imported coordinates. The service caches rounded
centroids in-process and treats provider failures as non-fatal to the
underlying geodata import; \`geocodeStatus\` records \`ENRICHED\`, \`FAILED\`,
\`NOT_CONFIGURED\`, or \`SKIPPED_NO_CENTROID\`.
