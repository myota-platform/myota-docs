# Phase 4 — client and operational migration

Status: implemented 2026-10-01

Phase 4 moves first-party clients to the preferred REST resources and makes
the migration observable. It does not remove compatibility aliases or change
programme policy. Deprecated routes remain available until the Phase 5 sunset
review.

## Client migration

The canonical contract repository now contains checked-in lightweight clients:

- [`contracts/typescript/myotaClient.ts`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/typescript/myotaClient.ts)
  is an authenticated-request façade for browser clients.
- [`contracts/python/myota_client.py`](https://github.com/myota-platform/myota-contracts/blob/main/contracts/python/myota_client.py)
  provides dependency-free typed methods for scripts and integration tests.
- [`check_generated_clients.py`](https://github.com/myota-platform/myota-contracts/blob/main/scripts/check_generated_clients.py)
  verifies preferred operation coverage in CI.

The Vue administration client uses `src/lib/myotaClient.ts` for programme,
identity, geodata, and award lifecycle writes. The public web client uses the
candidate proposal resource and the GET award-progress resource. Reads,
uploads, authentication commands, and genuinely asynchronous job resources
remain explicit where the contract requires them.

## Operational telemetry

Each HTTP service exposes `/metrics` in Prometheus text format. The
OpenTelemetry Collector scrapes those endpoints and receives OTLP telemetry;
Prometheus stores collected metrics and Tempo stores distributed traces. The
service metrics include:

- `myota_http_requests_total` by service, route, method, and status;
- `myota_legacy_route_requests_total` by deprecated alias;
- durable identity, programme, geodata, import, activity, QSO, participant and
  award aggregates;
- activity job gauges for queued, running, succeeded, and failed work;
- oldest queued activity-job lag; and
- pending QSO correction count.

Request counters reset with a service process, but durable business gauges are
read from PostgreSQL/PostGIS on scrape, so a restart does not erase the domain
view. OTLP exporters are asynchronous and best-effort; metrics never block API
writes when PostgreSQL or the collector is unavailable.

The dashboard files are maintained in
[`myota-deploy/observability`](https://github.com/myota-platform/myota-deploy/tree/main/observability)
and are packaged in the Helm chart when `observability.enabled=true`. Local
Compose enables them without changing the default stack:

```bash
docker compose --profile observability up -d prometheus grafana
```

Grafana is exposed at `http://localhost:3000`; Prometheus is exposed at
`http://localhost:9090`; the collector exporter is at `http://localhost:8889`;
and Tempo is at `http://localhost:3200`. The dashboard highlights real users,
entities, QSOs, participants, programmes, imports, awards, queue lag, HTTP
errors, OpenTelemetry request rates and p95 latency. It intentionally does not
use synthetic `vector(1)` or placeholder business values.

## Verification gate

The durable-stack verification script is
[`scripts/verify_phase4.sh`](https://github.com/myota-platform/myota-deploy/blob/main/scripts/verify_phase4.sh).
It runs the existing authorization, idempotency, audit, activity, award, and
geodata regression tests, checks typed-client coverage, and probes health and
metrics endpoints on the local Compose stack. Run it after a rebuild:

```bash
make verify-phase4
```

The implementation is intentionally additive: aliases still emit `Deprecation:
true` and the configured sunset date, while dashboard evidence is collected
before Phase 5 removes any route.
