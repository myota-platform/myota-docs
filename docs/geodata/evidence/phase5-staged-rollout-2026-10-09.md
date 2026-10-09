# Phase 5 staged rollout and operational evidence — 9 October 2026

## Scope and environment

This is a bounded operational qualification on the current provisional-production
K3s deployment, not a general production capacity claim. The one-node cluster
ran Kubernetes v1.35.8+k3s1. The deployed `myota-deploy` release was at commit
`a80257d36ac1d0046fd25b46bc6e0b1172902ef2`; the Geodata API used the
`ghcr.io/myota-platform/myota-geodata-service:latest` image, resolved on the
Ready pods to digest
`sha256:04202907b6a0e872f14aa86d9a98c7932e1c10f5b6f29fa47672b7baef4ba275`.
The starting API count was 2, with HPA range 2–3 at 70% CPU, `maxSurge: 1`,
`maxUnavailable: 0`, and a PDB with `minAvailable: 1`. The API Deployment has
no persistent-volume claim or pod-local upload volume; resumable upload data
is held by the shared SeaweedFS object-storage service.

All production write profiles used the existing guarded k6 harness, 50-VU
maximum, the dedicated `demo@example.test` account, exact run IDs, and enabled
run-scoped cleanup. No harness safety limits were changed. The account secret
and tokens are intentionally not recorded here.

## Live tests

### Accepted large upload during API rolling replacement

Run `lt-20261009-phase5-upload-recovery`: one VU, 5,000 features, 4,096 bytes
of bounded feature padding, 22,504,360 bytes total. The resumable upload,
checksum-verified part, and import acceptance checks all passed (4/4; 0 HTTP
failures). The run completed 9 requests; overall p95 was 2.86 s, control-path
p95 246.74 ms, and bulk-transfer p95 393.80 ms. The upload was accepted and
asynchronously processed while the two API pods were replaced one at a time.
The rolling update completed with two new Ready pods, the PDB remained in
force, and the gateway health endpoint returned HTTP 200. The processing
worker, PostGIS, SeaweedFS, and NATS were not restarted. Cleanup reported
`cleaned=true`, deleting 1 import, 1 object, and 1 upload session (0 entities).

This proves a 22.5 MB resumable upload/import can finish across an API rollout
on the same node. It does not prove node rescheduling, storage failure, or
recovery from terminating the worker or SeaweedFS during a production import;
those component restart cases remain in the isolated Phase 2/3 evidence.

### CPU-HPA scale-up during bounded concurrent edits

Run `lt-20261009-phase5-edit-rollout`: 50 VUs for 30 seconds against one
run-owned fixture entity. It completed 932 iterations and 944 HTTP requests.
Checks were 98.92% (922/932 concurrent-edit checks succeeded); HTTP failures
were 1.05% (10/944), and p95 control latency was 2.13 s, narrowly above the
harness's 2.00 s reporting threshold. The HPA observed 125% CPU against its
70% target and automatically requested replica 3; the new pod became Ready.
This is an actual CPU-triggered HPA event, unlike Phase 4's manual replica
boundary check.

The profile intentionally sends concurrent edits to the same entity. The
non-200 edit responses are retained as a same-row contention signal, not
silently counted as successful writes. This load shape is not representative
of independent editors. Service telemetry sampled a peak of 22 active
PostgreSQL connections on its observed series (100 configured maximum), a peak
of 1 pool-waiting request, and a peak of 22 sessions with PostgreSQL
`wait_event_type = 'Lock'`; the last is evidence of a lock-contention hotspot
under this deliberately colliding workload. The deployed scrape is not a
guaranteed sum across all API pod instances, so these gauges must not be read as
a cluster-wide connection total. Investigation of independent-row write
latency and lock attribution remains open.

Run `lt-20261009-phase5-edit-scaled`: repeated the same profile at three
replicas. It completed 942 iterations and 954 requests; 98.40% of checks
(927/942) succeeded, HTTP failures were 1.57% (15/954), and p95 control latency
was 1.91 s. Relative to the two-replica run, p95 improved by about 10.3%,
while observed throughput increased about 2.6% (23.58 to 24.19 HTTP requests/s).
Each run's cleanup removed its tagged import, one fixture entity, and one
source object. The small throughput change is not a broad capacity guarantee;
the shared-row contention and the threshold miss on the two-replica run remain
visible.

### Read-only catalogue/map comparison

At three Ready API replicas, the bounded 50-VU read baseline ran for 60 seconds
against the current 10-entity catalogue. It passed all 5,926 checks, made 5,925
requests (95.50 requests/s), and had 0% request failures. Overall p95 latency
was 15.98 ms. The retained two-replica 50-VU sample was 5,881 requests (94.77
requests/s), p95 18.76 ms, and 0% failures. The three-replica sample is about
0.8% higher in throughput and 14.8% lower in p95. This is a modest improvement
on a tiny catalogue, with different run windows; do not extrapolate it to the
2,875-entity historical scale fixture or a larger production catalogue.

The read checks covered health, catalogue pages, bounded map queries, and
sample entity details. The concurrent edit checks show most requests succeeded
but do not independently verify read-after-write ordering for separate
entities. No accepted upload depended on an API pod's local filesystem, and
all write fixtures were removed by the harness.

## Shared-service observations and cleanup

The post-run five-minute Prometheus window reported a maximum of zero for
JetStream consumer pending, ack-pending, redeliveries, and oldest-message age,
and zero SeaweedFS in-flight requests and PostgreSQL pool waiters. During the
write qualification window, the service-reported peak was 22 active PostgreSQL
connections, with a configured maximum of 100; the sampled peak lock-wait and
pool-wait counts were 22 and 1 respectively. The deployed scrape is not
guaranteed to sum all API replicas. The captured SeaweedFS S3 request rate is
not a representative peak-throughput figure; no object-store saturation claim
is made. After load, HPA CPU returned to 1% and
requested a reduction from three to two; the Deployment reported two desired
and Ready pods while the extra pod completed its 60-second termination grace.
The public gateway health endpoint returned HTTP 200.

Cleanup receipts:

| Run ID | Result |
| --- | --- |
| `lt-20261009-phase5-upload-recovery` | `cleaned=true`; 1 import, 1 object, 1 upload session deleted |
| `lt-20261009-phase5-edit-rollout` | `cleaned=true`; 1 import, 1 entity, 1 object deleted |
| `lt-20261009-phase5-edit-scaled` | `cleaned=true`; 1 import, 1 entity, 1 object deleted |
| 50-VU read-only comparison | No application data created |

## Qualification result

**Phase 5 remains open.** The evidence supports a bounded same-node rolling
replacement during a successfully accepted and processed large upload, an
actual HPA scale-up from 2 to 3 during concurrent edits, healthy reads during
the three-replica state, and exact cleanup. It does not establish a multi-node
reschedule, node/storage failure safety, end-to-end rollback, a sustained
large-import recovery across API plus worker/storage failure, broad independent
edit consistency, or a meaningful capacity gain beyond the tested data and
load. The single-node topology cannot provide independent node scheduling or
stateful-volume failure evidence. The same-entity lock-wait spike and 2.13 s
two-replica edit p95 also warrant further independent-row and larger-catalogue
measurement before treating the capacity objective as satisfied.

No code, chart, or runtime configuration change was required by this evidence;
the deployed HPA and rollout limits were left intact. The API roll and automatic
HPA action were themselves the live operational tests.
