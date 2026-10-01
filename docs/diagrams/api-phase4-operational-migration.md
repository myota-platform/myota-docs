# Phase 4 client and operations flow

```mermaid
flowchart LR
  Contract[Canonical OpenAPI contract] --> Py[Typed Python client]
  Contract --> Ts[Typed TypeScript client]
  Ts --> Admin[Admin web]
  Ts --> Public[Public web]
  Admin --> Gateway[Gateway]
  Public --> Gateway
  Gateway --> Services[Identity / Programme / Geodata / Activity]
  Services --> Metrics[Metrics endpoints]
  Metrics --> Prometheus[Prometheus]
  Prometheus --> Grafana[Operations dashboard]
  ActivityDB[(PostgreSQL activity_job)] --> Metrics
```

The compatibility aliases stay in the service route registries during Phase 4.
Their request counters allow the migration owner to identify remaining callers
before Phase 5 deprecation and removal.
