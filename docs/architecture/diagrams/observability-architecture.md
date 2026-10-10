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
  N[NATS JetStream]
  S[Surveyor - proposed]
  I --> C
  P --> C
  G --> C
  A --> C
  O --> C
  SW --> C
  O -->|read-only probes| SW
  O -->|current NATS samples| DB
  UI -->|current NATS status/history| O
  UI -->|Grafana auth subrequest| O
  O -->|live session and roles| I
  UI -->|per-user Editor or Viewer| GR
  C --> PR
  C --> T
  GR --> PR
  GR --> T
  N -. system monitoring requests .-> S
  S -. internal /metrics scrape .-> PR
```

The collector is the operational collection boundary for service telemetry.
The solid NATS-to-Operations history path is the current implementation; the
dashed Surveyor-to-Prometheus path is proposed and not deployed. After its
acceptance gates pass, Surveyor will replace broker-metadata polling while
Operations remains for SeaweedFS and Grafana identity integration. See the
[NATS monitoring consolidation plan](../../observability/nats-surveyor-migration.md)
and the [current legacy NATS status page](../../operations/messaging/jetstream-admin-status.md).
