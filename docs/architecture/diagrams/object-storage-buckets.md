# Object-storage bucket boundaries

All buckets share one configured S3-compatible endpoint. Bucket names are
independently configurable in Compose and Helm. The separation is for purpose,
access, and lifecycle policy; it does not imply a separate SeaweedFS cluster per
bucket.

```mermaid
flowchart LR
  Geo[Geodata service] -->|source files + snapshots| GeoBucket[(myota-geodata-imports)]
  Activity[Activity API + workers] -->|ADIF uploads| AdifBucket[(myota-adif)]
  AdifRetention[15-day terminal ADIF cleanup] -->|completed or failed source objects| AdifBucket
  Activity -->|editable backgrounds| BackgroundBucket[(myota-award-assets)]
  Activity -->|manager signatures| SignatureBucket[(myota-award-signatures)]
  Activity -->|issued certificate PDFs| CertificateBucket[(myota-certificates)]
  Retention[30-day geodata import cleanup] -->|only| GeoBucket
```

Background and signature bucket selection is server-side based on the asset
kind. Certificates remain durable issuance records and are never subject to
either import cleanup. ADIF cleanup deletes only source objects for completed
imports; import results remain in the activity database. Queued and processing
imports are excluded from the 15-day policy; both completed and failed imports
are eligible.
