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

Each HTTP service exposes `/metrics` in Prometheus text format. The counters
include:

- `myota_http_requests_total` by service, route, method, and status;
- `myota_legacy_route_requests_total` by deprecated alias;
- activity job gauges for queued, running, succeeded, and failed work;
- oldest queued activity-job lag; and
- pending QSO correction count.

Metrics are best-effort and in-memory for request counters. Durable activity
gauges are read from PostgreSQL on scrape, so a restart does not erase the
backlog view. Metrics never block API writes when PostgreSQL is unavailable.

The dashboard files are maintained in
[`myota-deploy/observability`](https://github.com/myota-platform/myota-deploy/tree/main/observability)
and are packaged in the Helm chart when `observability.enabled=true`. Local
Compose enables them without changing the default stack:

```bash
docker compose --profile observability up -d prometheus grafana
```

Grafana is exposed at `http://localhost:3000`; Prometheus is exposed at
`http://localhost:9090`. The dashboard highlights alias traffic, queue lag,
failed jobs, pending corrections, and HTTP errors.

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
