# Phase 5 read-only capacity follow-up — 11 October 2026

## Scope

This was a bounded read-only load check against the designated local K3s
provisional target, `https://api.myota.top`. DNS resolved to the K3s host. The
test used the main-branch `loadtests/geodata-baseline.js` script from
`myota-geodata-service` commit `ad28632723765463fa683ac175cf430045c769f0`;
the tested file's Git blob was
`c60558d599d4e1080e6c08cfea45d81a86489c15`. The deployed Geodata image was
`ghcr.io/myota-platform/myota-geodata-service@sha256:7599c09acecf9cbffd8d4f147af04ea2dc2ff95886464d1e08c047ef1daedd58`;
Helm release `myota` was revision 246, chart `myota-0.2.14`, at Deploy
`f7d2e65edd5248d70376cf99ccc98c836b7f95e3`.

The script exercised health, paged catalogue, bounded map, and sampled entity
detail requests. It sampled the current catalogue of ten entities. No API write,
fixture, test user, import, entity, object, or broker message was created; no
cleanup was required. No namespace, workload, HPA setting, or release was
changed.

## Results

Each profile ran for 60 seconds. The script passed its configured HTTP-failure
and check-rate thresholds.

| VUs | HTTP requests | Request failure rate | Checks | p95 request latency | Maximum request latency |
|---:|---:|---:|---:|---:|---:|
| 2 | 241 | 0% | 242/242 (100%) | 17.59 ms | 37.45 ms |
| 10 | 1,189 | 0% | 1,190/1,190 (100%) | 18.26 ms | 1.01 s |
| 50 | 5,901 | 0% | 5,902/5,902 (100%) | 22.94 ms | 2.07 s |

At 50 VUs, observed throughput was 95.07 HTTP requests/s. This is a one-minute
read profile on a ten-entity catalogue, not a sustained-load or broad-catalogue
capacity result. It does not substitute for independent-row write consistency,
large-catalogue query plans, upload/preprocessing/promotion load, or the earlier
two-versus-three replica write comparison.

## Post-run state

At 17:35:48 UTC, the Geodata API Deployment was 2/2 Ready and the processing
worker was 1/1 Ready. The HPA remained within its configured 2–3 range at 2
replicas and reported 36% average CPU against its 70% target. Node metrics were
18% CPU and 29% memory. A read-only catalogue query still returned ten total
entities. The gateway health check returned HTTP 200 before the profiles.

The run did not trigger automatic scale-up. These observations do not qualify
the HPA's write-workload behavior; the prior 9 October CPU-triggered scale-up
evidence remains the relevant write-profile result.

## Remaining Phase 5 gates

Phase 5 remains open. This follow-up cannot close the write and rollout gates:

- The exact-tag write profiles require credentials for the dedicated
  `demo@example.test` global test account. No such credentials were present in
  the local environment or secret files; the Kubernetes `myota-auth` Secret
  contains only the token-signing key. No credentials were read from unrelated
  secrets, created, or printed.
- The larger-catalogue and independent-row write profiles therefore were not
  run. The known same-row contention and two-replica p95 miss remain unresolved
  for that workload.
- No new Geodata image candidate was released, so a canary of a new image was
  not meaningful or run.
- Node loss, node drain, physical storage loss, and restarts of production
  PostgreSQL, SeaweedFS, or NATS remain prohibited by the roadmap. This
  single-node cluster cannot demonstrate multi-node scheduling or node-loss
  availability safely. Those limits are not recorded as passed.
- No end-to-end rollback, in-flight production import recovery, or production
  queue redrive was attempted.

**Disposition:** record the read-path result as additional evidence only. Keep
all unpassed Phase 5 exit criteria unchecked and do not increase the charted
replica ceiling or claim general scale qualification.

To continue the authorized write qualification, provide the existing dedicated
`demo@example.test` account credentials through the protected local secret
path, plus a candidate image digest if the canary gate is to be exercised.
Node-loss qualification requires a disposable multi-node test environment;
it must not be simulated by draining this host's only node.
