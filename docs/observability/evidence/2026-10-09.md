# Storage observability and Grafana delivery verification — 9 October 2026

Environment: live K3s on `spainip.es`; Admin UI at `https://admin.myota.top`.
Date uses Europe/Madrid; the Helm deployment timestamp is 8 October UTC.
Credentials and tokens were read only inside the host verification process
and were neither printed nor stored in this record.

## Delivered commits

| Repository | Commits |
| --- | --- |
| Operations service | `1b78e9d` native storage probes/migration; `476adc5` authenticated status/history and current-role Grafana identity |
| Admin web | `769ba2d` storage page; `762ec44` trusted per-user Grafana proxy headers |
| Platform | `fbd3940` synchronized operations, schema, contract and gateway integration |
| Deployment | `b946cb5` editable dashboard/default provisioning; `feb0942` Compose/Helm/service integration; `fb5b655` image digest rollout |
| Contracts | `16b0677` three operations resources, client methods and frozen inventory |
| Docs | `91ae7e2` runbook, indexes, architecture and initial timeline; this record supplements the deployed verification |
| Organization profile | `eca8224` operations responsibilities and completed checklist items |

## Local and CI verification

- Operations: 10 tests passed, including HTTP header propagation, current-role
  Editor/Viewer mapping, unauthorized and revoked-session denial, real metric
  parsing, null missing gauges, truncation and sanitized provider failure.
- Admin UI: TypeScript check, production Vite build and 20 regression tests passed.
- Platform: 34 tests completed, with the optional isolated-NATS test skipped
  because no dedicated test broker was supplied. The remaining 33 passed.
- Deployment: five observability configuration tests passed. All five source
  and Helm dashboard JSON documents match, are editable, use `now-30m`/`now`
  and refresh every `30s`.
- Contract reconciliation: 167 operations, 266 runtime registrations, no
  missing registrations/contract routes or duplicate operation IDs. YAML,
  synchronized mirrors, inventory and client coverage checks passed.
- [Platform CI](https://github.com/myota-platform/myota-platform/actions/runs/37850360238),
  [operations quality/image](https://github.com/myota-platform/myota-operations-service/actions/runs/37850367179),
  [Admin web build/image](https://github.com/myota-platform/myota-admin-web/actions/runs/37850372976),
  [Helm rendering](https://github.com/myota-platform/myota-deploy/actions/runs/37850624399),
  [deployment images](https://github.com/myota-platform/myota-deploy/actions/runs/37850624372),
  and [contract freeze](https://github.com/myota-platform/myota-contracts/actions/runs/37850635601)
  completed successfully. Helm rendering was performed in GitHub, not locally.

## Live results

- Helm chart `myota-0.2.11`, release revision **97**, status **deployed**.
  Fleet reported **1/1 ready** at Git commit `fb5b655`. All 21 MyOTA deployments
  had their desired Ready replicas.
- The core schema readiness marker was true (migration revision 96). The
  operations-owned `operations_storage_snapshot` table was present with 13
  durable rows at the later verification. No existing data was removed.
- Authenticated latest/history APIs through the Admin UI proxy returned a
  fresh **HEALTHY** sample, one reported bucket and 10 reported objects. History
  pagination returned recorded samples. Unknown fields remained nullable.
- Operations `/metrics` returned `myota_operations_object_storage_up 1` and
  the real last-sample timestamp gauge.
- The real `demo@example.test` GLOBAL_OPERATOR account resolved to a distinct
  Grafana identity and its current organization role was **Editor**.
  `isGrafanaAdmin` was false. Spoofed `X-WEBAUTH-USER`/`ROLE` headers were ignored.
- All five Grafana dashboards returned the requested time range and refresh
  defaults; `meta.canEdit` was true for the authenticated operator.
- The operator successfully created a temporary dashboard containing a
  Prometheus timeseries panel, retrieved the saved panel, then deleted the exact
  temporary UID `myota-check-1a4231c5e2f74f08abde`. It was test-only data.
- Anonymous storage API access returned **403**; anonymous Grafana API access
  redirected to login (**302**).

No signed-in browser tab was available for a visual review of the Vue page.
Build/type checks and real proxy/API/Grafana integration were verified;
the manual visual UI check remains unperformed. Existing Identity/Programme
scrape gaps and the broader Loki/structured-logging roadmap were not qualified
by this delivery.
