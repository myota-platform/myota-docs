# Geodata TracerProvider initialization repair

**Verified:** 11 October 2026  
**Scope:** Geodata OpenTelemetry initialization, synchronized runtime copies,
immutable image rollout, and a safe live trace check on host K3s.

## Root cause

The warning came from two code paths trying to set the process-global tracing
provider:

1. `run_geodata.py` calls `configure_logging("myota-geodata")` at startup.
   `myota_logging.configure_logging()` calls
   `myota_logging.configure_tracing()`, which installs an SDK `TracerProvider`.
2. On each request, `common.JsonHandler._dispatch()` calls
   `telemetry_for(self.service)`. Before this repair,
   `otel.telemetry_for()` created and registered another `TracerProvider`,
   cached only by service name, and could repeat that registration for a
   different name.

This is confirmed by the startup and request call sites in
`run_geodata.py`, `common.py`, `myota_logging.py`, and `otel.py`. The regression
test exercises that startup-then-request order in a fresh Python process and
asserts exactly one global provider registration.

## Changes

- `myota_logging.configure_tracing()` now reuses an installed SDK provider,
  attempts process setup once, and reports tracing or PostgreSQL
  instrumentation setup failures using the exception class only. It does not
  include endpoint values or exception text in those diagnostics.
- `otel.telemetry_for()` no longer creates or registers a tracing provider. It
  uses the global API tracer established during startup and returns one cached
  telemetry object for the process, regardless of later service-name inputs.
- Metrics setup remains best-effort. Setup errors produce a bounded warning
  and no-op metrics while request handling remains available.
- The Compose and Helm audit found no external `opentelemetry-instrument`
  wrapper or competing Geodata provider setup. Compose builds the Geodata
  owner source; Helm runs its regular API/import-worker entrypoints and sets
  the documented OTel environment variables. No deployment configuration
  change was needed beyond immutable image digests.
- The Platform runtime logger is authoritative. Its updated implementation
  was synchronized to Identity, Programme, Activity, Geodata, Operations,
  Deploy, and Platform logger mirrors. Geodata's `otel.py` was synchronized to
  Deploy and Platform runtime mirrors.

## Verification

- Geodata focused tests: 9 tests passed across `tests.test_otel_metrics`,
  `tests.test_myota_logging`, and `tests.test_shared_compatibility`.
- Ruff check, Ruff format check, Python compilation, and `git diff --check`
  passed for the changed source/tests. `scripts/sync_runtime_logging.py
  --check` passed for all seven shared-logger mirrors.
- Geodata Python quality workflow passed:
  [run 38170229556](https://github.com/myota-platform/myota-geodata-service/actions/runs/38170229556).
- Geodata image build passed and published
  `ghcr.io/myota-platform/myota-geodata-service@sha256:6b8fc171db83883da8cad0fc8739b6188d07777f3f3bb071fd09e443e5c27111`:
  [run 38170228835](https://github.com/myota-platform/myota-geodata-service/actions/runs/38170228835).
- Deploy Python quality and shared-runtime image publication passed:
  [quality 38170319192](https://github.com/myota-platform/myota-deploy/actions/runs/38170319192),
  [images 38170318770](https://github.com/myota-platform/myota-deploy/actions/runs/38170318770).
- Identity, Programme, Activity, and Operations shared-logger mirror/image
  workflows passed: Identity [quality](https://github.com/myota-platform/myota-identity-service/actions/runs/38170320906)
  and [image](https://github.com/myota-platform/myota-identity-service/actions/runs/38170320321),
  Programme [quality](https://github.com/myota-platform/myota-programme-service/actions/runs/38170317884)
  and [image](https://github.com/myota-platform/myota-programme-service/actions/runs/38170317159),
  Activity [notification tests](https://github.com/myota-platform/myota-activity-service/actions/runs/38170325827)
  and [image](https://github.com/myota-platform/myota-activity-service/actions/runs/38170325353),
  and [Operations quality/image](https://github.com/myota-platform/myota-operations-service/actions/runs/38170322918).
- Platform's workspace CI had unrelated existing failures: the event-registry
  test expects 67 entries while the synchronized registry has 68, and Ruff
  reports formatting in `services/identity.py`. The shared-logger mirror audit
  passed; those unrelated issues are not changed here.

## Host K3s rollout and trace check

The only live request used for the trace check was a read-only `GET /healthz`.
It created no test records or other persistent domain data. The trace was
queried from the in-cluster Tempo HTTP API and inspected for bounded attributes
only. The verified trace was `3aa1681d0d31d7deca8893e8faa580e7`; its resource
was `service.name=myota-geodata`, and its span contained only method `GET`,
route `/healthz`, and status `200` as request attributes.

The final host rollout is Helm release `myota`, chart `myota-0.2.14`,
revision `257`, status `deployed`. Fleet's Deploy GitRepo is Ready 1/1 at
`f37a1fa8b5f3f5c0eb639c7125fcc9ab0fe323b8`, and the MyOTA bundle is deployed
and monitored. All 23 Deployments report their desired replicas ready (27/27
replicas). Both Geodata API replicas and the import-processing worker use
`ghcr.io/myota-platform/myota-geodata-service@sha256:6b8fc171db83883da8cad0fc8739b6188d07777f3f3bb071fd09e443e5c27111`.
The runtime logger image is pinned to
`ghcr.io/myota-platform/myota-service@sha256:8050be67423e234c2fad94e5beb6179e34bb692d0b49274d30dc5f7966333f0b`.

The exact provider-override warning and tracing/metrics setup errors were
absent from Geodata pod logs after rollout. The read-only `GET /healthz`
returned HTTP 200 and its trace was present in Tempo. No test records,
temporary Kubernetes resources, or persistent test data were created.
