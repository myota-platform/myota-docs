# Current client and operations flow

~~~mermaid
flowchart LR
  Contract[Canonical OpenAPI] --> Py[Python client facade]
  Contract --> Ts[TypeScript client facade]
  Ts --> Admin[Admin web]
  Ts --> Public[Public web]
  Admin --> Gateway[Gateway]
  Public --> Gateway
  Gateway --> Services[Identity / Programme / Geodata / Activity]
  Services --> Metrics[Service metrics]
  Metrics --> Prometheus[Prometheus]
  Prometheus --> Grafana[Current dashboards]
  NATS[NATS server] --> Surveyor[NATS Surveyor]
  Surveyor --> Prometheus
~~~

Contracts owns the client facades and API definition. Service repositories own
handlers; Deploy owns the runtime image and chart; Platform is an integration
mirror. Grafana dashboards use service metrics and the Surveyor scrape for
broker monitoring. No alias-specific request counter is part of current
service metrics.
