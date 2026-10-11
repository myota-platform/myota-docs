# NATS Surveyor migration evidence — 11 October 2026

## Source changes

- `myota-operations-service`: commit
  `601b9fc42013e98c8531320185b08f4fb756f940`; CI passed. NATS inspection,
  sampler, API and dependency were removed. Operations storage history and
  Grafana identity remain.
- `myota-admin-web`: commit
  `8c63f9ddc2f50091e975c836e5413e2bd6405c26`; CI passed. The broker page,
  route, client, styles and obsolete test assertion were removed.
- `myota-geodata-service`: commit
  `2a73101a741b8325afba9cda9176be0d85003e65`; Python quality passed. The
  independent broker metrics poller and startup hook were removed.
- `myota-deploy` owns Helm/Compose and
  `myota-platform` mirrors them. Both carry Surveyor 0.9.11, the 11 dashboards,
  bounded store config, probes, 15-second scrape, 15-day Prometheus retention
  and alert thresholds. The Surveyor source dashboards are pinned to
  `nats-io/nats-surveyor@0539d22d7bc7a4574c7f376f1dde37f00bfe2ee9`.

## Live checks before corrected rollout

- NATS Surveyor was Ready, connected to the internal NATS Service, and exposed
  its metrics endpoint. Prometheus returned `up{job="nats-surveyor"}=1` and
  `nats_up=1`.
- JetStream reported 120.56 GiB host-derived file-store capacity and a
  4 GiB reservation. File and memory store use were both zero. The PVC is 8 GiB
  local-path and does not enforce a filesystem quota; the source cap is 7 GiB
  file and 1 GiB memory.
- Grafana previously listed 11 dashboards in folder NATS. 16256's adapted
  metrics were present. A metric-family audit found 48 of 78 distinct NATS
  series names referenced across the set. Missing families are for absent
  routes/gateways, multi-server Raft/meta-layer state, and optional latency
  observations. Those panels are not applicable to this single-server
  deployment and must display no data, not zero.
- An earlier read-only test used an in-memory JetStream stream; it was deleted
  afterward. Current file-store use is zero and no test pod is retained.
- A custom-format backup of 9,657 Operations snapshot rows was created outside
  source control, mode 0600, and verified with `pg_restore --list`. SHA-256:
  `634520fef35d62e647eb34b01d803efa7d77a91bf5e43eba7eda65118e215428`.

## Rollout correction and remaining gates

The first Helm render converted numeric byte limits to scientific notation.
NATS rejected the replacement config before starting. The old NATS pod remained
Ready, and `nats_up=1`. Helm revision 202 became pending; revision 203 failed
its readiness check due to Kubernetes API client rate limiting; revision 204
restored revision 201 without waiting. Deploy commit
`221c44ccb0b1c5fe6ea894c7a2c4589be7685cd9` now stores limits as decimal
strings and sets the single-replica NATS Deployment to `Recreate`. Fleet is
being asked to reconcile this fixed source.

The replacement images are not yet confirmed live. The snapshot table remains.
Apply an Operations-owned forward migration only after its image has rolled
out; preserve the verified backup. Finish by checking Fleet Ready, all eleven
Grafana dashboards and query errors, exporter/server health, NATS caps, absence
of the old Admin route/Operations metrics/Geodata poller, storage and Grafana
identity paths, and absence of the migrated snapshot table/index. This record
must be updated with final live evidence before any checkbox is marked complete.

## Final live verification — 11 October 2026

- Deploy main supplied the corrected chart and the migration image digest.
  Helm revision 206 completed; the replacement Operations, Admin, and Geodata
  images are Ready at the pinned digests in Deploy main. NATS logs report a
  7,516,192,768-byte file-store cap and 1,073,741,824-byte memory-store cap.
  The single NATS pod recovered the existing three streams and eleven durable
  consumers without changing their state.
- After restarting the stateless exporter, Prometheus reported one active NATS
  server ID, `up{job="nats-surveyor"}=1`, `nats_up=1`, and the configured
  7,516,192,768-byte file-store capacity. The earlier server-ID series expired.
  Surveyor exposed 84 metric names. The compatibility audit still classifies
  absent route/gateway, multi-server Raft, and optional latency series as
  unsupported for this single-server topology.
- Grafana API search returned all eleven dashboards in folder `NATS` and no
  `MyOTA JetStream backlog` or `PostGIS query performance` dashboard. Dashboard
  16256 and the ten pinned Surveyor dashboards are provisioned from Deploy.
- A disposable `nats-box` pod created the memory-only stream
  `MYOTA_SURVEYOR_VALIDATION_20261011` (three-minute maximum age), published one
  synthetic 53-byte payload, and confirmed Surveyor exported its stream label.
  The stream and pod were deleted immediately afterward. No validation stream or pod
  remains. Prometheus's normal 15-day metrics retention is unchanged.
- Operations migration 003 is mirrored as Deploy/Platform core migration 005.
  The `myota-migrations-207` Job completed. Database inspection confirmed
  `operations_jetstream_snapshot` and its history index are absent, while
  `operations_storage_snapshot` contains 6,031 rows at the final check. The pre-migration custom
  format backup remains outside source control with SHA-256
  `634520fef35d62e647eb34b01d803efa7d77a91bf5e43eba7eda65118e215428`.
- Fleet reports Ready=True for Deploy main commit
  `fa4e0127af2cc7d29851f1f76b3bb8303655ad46`. Helm revision 207 is deployed,
  the bundle applied/desired deployment IDs match, and 64/64 resources are
  ready. The earlier `WaitApplied(1)` state cleared after the local Fleet agent
  restart and one force sync.
- Focused CI: Operations migration commit `9dd93091b9abcb74fb5c047bbbd6a592edffd186`
  passed run `38097580535`; Deploy migration commit
  `bada0388a0e684f8fea87b15452f7df87eced170` passed Python quality run
  `38097585028`; its runtime image build passed run `38097584634`.
