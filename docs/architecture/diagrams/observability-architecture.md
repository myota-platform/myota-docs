# Observability architecture

```mermaid
flowchart LR
  UI[Admin UI]
  Ops[Operations service]
  DB[(Core PostgreSQL)]
  SW[SeaweedFS]
  NATS[NATS JetStream]
  Surveyor[NATS Surveyor]
  Prom[Prometheus]
  Grafana[Grafana]
  UI -->|Grafana auth subrequest| Ops
  Ops -->|read-only storage status| SW
  Ops -->|storage history only| DB
  NATS -->|cluster-local system monitoring| Surveyor
  Surveyor -->|15 second scrape| Prom
  Prom --> Grafana
```

Operations no longer needs a NATS connection for broker metrics. Surveyor is
the central broker observer; its Service and NATS connection stay internal.
The eleven dashboards (dashboard 16256 plus ten pinned Surveyor dashboards)
are provisioned in Grafana folder `NATS`. Prometheus keeps 15 days of metrics,
not broker messages. The single-server deployment leaves cluster, gateway,
multi-node Raft, and optional service-observation panels without data.


**Implementation status (11 October 2026):** Surveyor 0.9.11, the eleven NATS
folder dashboards, bounded JetStream capacities, and replacement Admin,
Operations, and Geodata images are live. Helm revision 206 applied the
Operations snapshot retirement migration; storage history remains. The
remaining gate is Fleet reconciliation: `myota-deploy` reports
`WaitApplied(1)` with stale applied-release metadata despite the healthy Helm
release and workloads. See the [NATS monitoring roadmap](../../observability/nats-surveyor-migration.md)
and [live evidence](../../observability/evidence/2026-10-11-nats-surveyor-cutover.md).
