# Proposed MyOTA repositories

The following split is justified and intentionally small:

| Repository | Owns | Initial source here |
|---|---|---|
| `myota-contracts` | OpenAPI, event schemas, compatibility rules, generated client release | `contracts/` |
| `myota-identity-service` | accounts, callsigns, auth claims, OIDC mappings | `services/identity.py`, core migrations |
| `myota-programme-service` | programmes, entity types, programme-owned rules and themes | `services/programmes.py`, core migrations |
| `myota-geodata-service` | PostGIS, import adapters, provenance, conflation, review | `services/geodata.py`, geo migrations |
| `myota-activity-service` | activations, QSOs, activity-derived award progress, programme-owned award definitions, object-storage assets, requests, rendering and issuance records | `services/activity.py`, `services/awards.py`, core migrations |
| `myota-web` | universal programme UI, published award progress and participant requests | `web/` |
| `myota-admin-web` | authenticated global administration web, programme context, review queues, award designer, asset management and operational views | `myota-admin-web/web/` |
| `myota-deploy` | Helm charts, environments, migrations, observability | `deploy/`, `compose.yaml` |
| `myota-docs` | architecture, ADRs, operator and migration docs | `docs/` |

The bootstrap repository is a temporary integration workspace; it is not a reason to create many more repositories. Once the MyOTA organization is available, each row can be created from the corresponding paths and wired together by pinned contract versions.
