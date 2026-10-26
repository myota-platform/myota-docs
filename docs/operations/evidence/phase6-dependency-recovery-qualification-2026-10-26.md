# Phase 6 dependency-recovery qualification — 2026-10-26

This record consolidates deployed, single-instance recovery evidence for the
isolated `myota-test` release. It covers NATS/JetStream, the three service-owned
PostgreSQL databases, and SeaweedFS object storage. It does not claim HA,
multi-instance failover, or recovery after permanent volume loss.

Related work: [Phase 6 issue #10](https://github.com/myota-platform/myota-next/issues/10),
[Activity recovery #15](https://github.com/myota-platform/myota-next/issues/15),
[Geodata recovery #16](https://github.com/myota-platform/myota-next/issues/16),
[NATS and PostgreSQL recovery #17](https://github.com/myota-platform/myota-next/issues/17),
and [backup/restore #89](https://github.com/myota-platform/myota-next/issues/89).

The detailed test procedure, fixture cleanup, readiness observations, and
per-operation results are in the
[MyOTA Next recovery record](https://github.com/myota-platform/myota-next/blob/testing/docs/operations/single-instance-restart-recovery.md).
The deployed NATS/PostgreSQL and Geodata/Activity recovery packages were
verified and closed in issues #15–#17.

## Evidence summary

- **PostgreSQL:** Core, Activity, and Geodata were interrupted separately
  while transactions held uncommitted markers. Committed markers remained and
  uncommitted markers rolled back. Deployed APIs, relays, and workers returned
  to Ready after their respective database endpoints recovered. Database
  ownership remained separate.
- **NATS/JetStream:** A file-backed durable redelivered an unacknowledged
  message after NATS restarted. Pending and acknowledgement-pending counts
  returned to zero after acknowledgement. Core outbox work remained retryable
  during an outage and appeared once in the retained stream after recovery.
- **Business work:** Deployed QSO-ingestion, award-recalculation, Geodata
  preprocessing, Activity notification, and Email work was exercised through
  database, NATS, or worker interruptions. Recorded outcomes preserve one
  committed business result, use retry or lease recovery where applicable, and
  settle the corresponding durable after recovery.
- **Object storage:** Deployed ADIF upload and worker reads, profile images,
  Geodata multipart sessions/imports, and award PDF output were exercised
  during SeaweedFS outages or injected S3 failures. Retries completed without
  duplicate QSO or artifact results. A source object also remained readable
  after SeaweedFS pod replacement with its PVC retained.
- **Cleanup:** The test record reports removal of the unique probe rows,
  objects, imports, events, triggers, and other fixtures after each check.

## Current topology spot check

On 2026-10-26, Helm reported `myota-test` revision 26 as deployed. The three
PostgreSQL StatefulSets, single NATS StatefulSet, and SeaweedFS StatefulSet
were Ready; their five PVCs were Bound. This was a read-only topology check,
not a new fault-injection run. The release remains a single-instance dependency
topology on a single-node K3s server.

## Limits and remaining work

- These checks demonstrate process restart and transient dependency recovery;
  they do not cover permanent local-PVC loss or prove a backup restore. Rehearse
  database and object-store backup/restore under [issue #89](https://github.com/myota-platform/myota-next/issues/89).
- The evidence does not cover clustered NATS, HA PostgreSQL, or horizontal
  service failover. Those remain outside Phase 6 under
  [future infrastructure issue #24](https://github.com/myota-platform/myota-next/issues/24).
- Identity and Geodata exposed some object-store connection failures as HTTP
  500 even though their persisted state remained safe and a caller retry
  succeeded. Preserve this error-mapping gap in the remaining-work ledger.
- A Geodata lease-loss warning was observed after its transaction had already
  committed; the worker safely observed terminal owner state and settled the
  message. The recovery record documents this race for follow-up.

