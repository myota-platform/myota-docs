# Trail and way geometry

MyOTA represents a trail, route, boundary-less path, or other linear geodata entity as a GeoJSON `LineString`. The administration UI labels this geometry kind **Way / trail** so programme administrators do not need to know the GeoJSON term.

## Import format

Standard GeoJSON is preferred:

```json
{
  "type": "Feature",
  "properties": {
    "name": "Sendero de ejemplo",
    "entityType": "TRAIL"
  },
  "geometry": {
    "type": "LineString",
    "coordinates": [[-5.99, 37.39], [-5.98, 37.40]]
  }
}
```

For OSM-style adapter records, the service also accepts `type: "way"` with a top-level `coordinates` array. It is normalized to a GeoJSON `LineString` before validation and persistence. A way must contain at least two WGS84 longitude/latitude positions.

## Administration workflow

- Select **New candidate → Way / trail**, start drawing, and finish the line on the map.
- Select an existing way and use **Edit geometry** to move vertices; saving creates the normal geometry audit entry.
- GIS or global administrators can change the geometry kind between Point, Way / trail, and Polygon. Point-to-way and polygon-to-way conversions are deterministic convenience conversions and should be reviewed before approval.
- The map renders ways as linear features and fits the viewport to the full trail extent when selected.

The programme still owns the meaning of the entity type, eligibility rules, and activation policy. MyOTA provides only the reusable geospatial representation and review workflow.
