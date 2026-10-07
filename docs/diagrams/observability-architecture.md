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
  I --> C
  P --> C
  G --> C
  A --> C
  O --> C
  C --> PR
  C --> T
  GR --> PR
  GR --> T
```

The collector is the single operational collection boundary. Business facts
remain service-owned; observability infrastructure does not become a new
source of truth.

The [JetStream admin page](../jetstream-admin-status.md) reads protected APIs
from the operations service. Broker snapshots are recorded in its control-plane
table; business-domain workers remain separate. The operations service is an
observer, never a consumer, ACK source, queue executor or message archive.
