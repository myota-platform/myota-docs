# Observability architecture

```mermaid
flowchart TB
  subgraph Services
    I[Identity /metrics and OTLP]
    P[Programme /metrics and OTLP]
    G[Geodata /metrics and OTLP]
    A[Activity /metrics and OTLP]
    O[Operations /metrics and OTLP]
  end
  C[OpenTelemetry Collector]
  PR[Prometheus]
  GR[Grafana]
  T[Tempo]
  SW[SeaweedFS private health and metrics]
  UI[Admin UI and trusted Nginx proxy]
  DB[(myota_core sampled history)]
  I --> C
  P --> C
  G --> C
  A --> C
  O --> C
  SW --> C
  O -->|read-only probes| SW
  O -->|samples| DB
  UI -->|status and history APIs| O
  UI -->|Grafana auth subrequest| O
  O -->|live session and roles| I
  UI -->|per-user Editor or Viewer| GR
  C --> PR
  C --> T
  GR --> PR
  GR --> T
```

The collector is the single operational collection boundary. Business facts
remain service-owned; observability infrastructure does not become a new
source of truth.

The [JetStream admin page](../../operations/messaging/jetstream-admin-status.md) reads protected APIs
from the operations service. Broker snapshots are recorded in its control-plane
table; business-domain workers remain separate. The operations service is an
observer, never a consumer, ACK source, queue executor or message archive.
