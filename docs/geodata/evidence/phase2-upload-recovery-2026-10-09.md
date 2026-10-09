# Phase 2 resumable-upload recovery evidence — 9 October 2026

## Result

**Passed. Phase 2 is complete for the tested SeaweedFS image digest.** The
production K3s SeaweedFS pod was inspected read-only to obtain its runtime image
ID. The destructive restart exercise then ran only in an isolated GitHub
Actions job, using that same immutable image digest and disposable PostGIS and
SeaweedFS data.

| Item | Evidence |
| --- | --- |
| Deployed SeaweedFS image ID observed in K3s | `docker.io/chrislusf/seaweedfs@sha256:4e61d15fd35994cb1e43e1e553dff106794841fd9a99ade2fc8c8bfce4d7872d` |
| Geodata service revision under test | [`853fcbc`](https://github.com/myota-platform/myota-geodata-service/commit/853fcbc5b30085c8a450ff0accbe32e414348c22) |
| Automated recovery and service quality run | [GitHub Actions run 37908154059](https://github.com/myota-platform/myota-geodata-service/actions/runs/37908154059) |
| Restart-recovery test | One end-to-end scenario passed in 14.944 seconds; isolated job passed in 52 seconds |
| Related regression/quality checks | Relational-boundaries job passed (including migration, Ruff format/lint, and service regressions); image publication succeeded |
| Test data disposition | Disposable database service and named SeaweedFS volume removed at job end |
| Production effect | No production write, API termination, SeaweedFS restart, or production object cleanup |

## Scenario and assertions

The scenario used two bounded parts: 5 MiB and 1 MiB + 137 bytes, for a total
of 6,291,593 bytes. It created an owner-bound upload session with the expected
whole-object SHA-256, uploaded the first part, terminated the API process, and
started a fresh API process against the same PostGIS database and SeaweedFS
multipart upload.

The new API recovered the session and first-part checksum from durable state.
Replaying that part left a single part record. A different subject could not
read the session, resume by sending a part, or abort it. After the second part
was acknowledged, the isolated SeaweedFS container was restarted while
preserving its test-only persistent volume. The API still reported both
parts, completed the multipart object, and returned the expected SHA-256.

The test independently read the stored S3 object and recomputed the whole-file
checksum. It also checked that the completed upload row and import source
metadata persisted, and that exactly one import outbox event existed after a
completion retry. A second incomplete session was aborted and verified to
have no remaining multipart upload. CI then removed the disposable object-store
volume and database container.

The uploaded bytes are a deterministic transfer fixture; the test deliberately
does not enqueue parsing work. It proves resumable transport and durable
handoff metadata, not GeoJSON validity, preprocessing, parser memory bounds,
worker processing, node/disk failure recovery, or SeaweedFS cluster failover.

## Repeatability and image changes

The test is `tests/integration_upload_session_recovery.py` in
[`myota-geodata-service`](https://github.com/myota-platform/myota-geodata-service/blob/main/tests/integration_upload_session_recovery.py).
The `test-upload-session-recovery` job in
[`build-and-publish.yml`](https://github.com/myota-platform/myota-geodata-service/blob/main/.github/workflows/build-and-publish.yml)
starts a fresh PostGIS database, starts the digest-pinned SeaweedFS image with
a disposable named volume, applies service-owned migrations, runs the test,
and deletes its object-store container/volume even on failure. It is part of
the required image-build checks, not a production test.

When the live K3s SeaweedFS pod reports a different image ID, update the
workflow's immutable image reference and this evidence after confirming that
new image in the deployment. The workflow must not switch to a floating tag.
This one-digest recovery result does not qualify arbitrary future SeaweedFS
versions.

## Phase status

The Phase 2 exit gate is closed for the recorded image digest. Phase 3 remains
open for bounded-memory parsing and worker failure-injection/recovery. Phase 5
still requires broader receiver-pod, worker, duplicate-delivery, storage/node
failure, canary and rollback evidence. See the
[horizontal-scaling roadmap](../horizontal-scaling-roadmap.md) and the
[Phase 2 test specification](../prompts/scaling/phase-2-upload-recovery.md).
