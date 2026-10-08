# Phase 2 prompt — upload recovery across restarts

Copy the prompt below into a coding task with the MyOTA repositories available.

---

You are completing **Phase 2 — make upload handoff durable without a shared pod
volume** of the MyOTA Geodata API horizontal-scaling roadmap. Start with these
pages and inspect the live implementation before changing anything:

- `myota-docs/docs/geodata-horizontal-scaling-roadmap.md`
- `myota-docs/docs/admin-web-ux.md`
- `myota-docs/docs/geodata-load-test-upload-verification.md`
- `myota-docs/docs/repository-map.md`

The current project premise sends upload/load-performance profiles to the
provisional-production API. This Phase 2 prompt is specifically about
destructive interruption and recovery: those checks must remain in CI or an
isolated non-production environment and must never restart the live production
API or SeaweedFS service.

The resumable upload-session API, bounded 16 MiB part handling, durable object
handoff, browser pause/resume/discard, checksum verification, and removal of
the shared upload-spool PVC are implemented. The remaining gate is to prove
session recovery through API termination and object-storage restart against
the production SeaweedFS image/version used by deployment.

Inspect the owning service, deployment manifests, image pin/version, upload
state transitions, and existing integration tests. Add or adapt an automated
integration/failure-injection test that exercises the deployed SeaweedFS image
in an isolated non-production environment. It must cover at minimum:

1. Create an upload session and persist multiple bounded parts.
2. Terminate/restart the API between parts, then authenticate as the same owner
   and resume from the persisted part list/checksums.
3. Restart the SeaweedFS service at a controlled point and verify the session
   recovers or fails visibly and safely, without falsely accepting incomplete
   bytes.
4. Complete the upload and verify the assembled object checksum and durable
   import metadata/outbox handoff.
5. Retry uncertain part/complete responses and verify idempotency; abort or
   expire an incomplete session and verify cleanup removes the multipart state
   and metadata safely.
6. Verify a different user cannot inspect, resume, or discard the session.

Use the exact deployment image/version, not a floating substitute. Prefer a
repeatable CI or isolated integration setup; never disrupt a shared or
production object store. Keep credentials out of source, logs, and artifacts.
Preserve bounded memory behavior and the no-shared-PVC architecture. If the
image is unavailable or CI cannot safely restart the service, build the test
and wiring that can run in an isolated environment, document the unmet runtime
evidence, and leave the gate open.

Fix implementation defects exposed by the test. Keep service-owned migrations
and their platform/deploy mirrors synchronized if schema changes are necessary.
Update the roadmap and upload runbook with the image digest, scenario, result,
and recovery behavior. Check off the restart gate only when the test passes
against the deployed image/version.

At the end, report the image/version tested, restart/failure scenarios,
checksum and cleanup results, changed files, validation performed, and any
remaining gate. Do not call Phase 2 complete based on the existing pinned local
image test alone.

---
