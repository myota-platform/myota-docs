# Resumable load-test uploader verification

Verified locally on 2026-10-07. This is protocol/correctness evidence, not a
production capacity result. No production write load was started for this fix.

## Failure and correction

The failed `large-upload` run used the old multipart
`POST /v1/geodata/imports/upload`. Durable deployments reject that route and
require the resumable API. Idle looping iterations made the output misleading:
the reported iteration count did not represent actual upload attempts.

The profile now runs one real upload per VU with these existing contract routes:

1. `POST /v1/geodata/import-uploads`: tagged metadata, size, SHA-256 and an
   idempotency key; require HTTP 201 and a session ID.
2. `POST /v1/geodata/import-uploads/{uploadId}/parts/{partNumber}`: binary body
   with `X-Part-SHA256`; check HTTP 200, byte total and returned checksum.
   Client parts never exceed 16 MiB; non-final parts meet S3's 5 MiB minimum.
3. `POST /v1/geodata/import-uploads/{uploadId}/complete`: require HTTP 202 and
   nested `importRun.id`. After an uncertain response, check session progress
   before attempting an abort. Failed HTTP requests still count as failures.
4. `DELETE /v1/geodata/import-uploads/{uploadId}`: abort incomplete transfers.
5. Teardown calls the exact-tag `DELETE /v1/geodata/load-test-runs/{testRunId}`.

No API route or database migration was added. Cleanup now removes terminal
upload-session rows and cascades their part metadata, alongside existing
tagged imports/entities/objects. Upload and import references to the same
object are deduplicated. Other run tags and active transfers are preserved;
the response adds `uploadSessionsDeleted`. Database cleanup uses the owning
request transaction and locks the tagged sessions before checking status.

## Reproducible checks and results

Run from `myota-geodata-service`:

```bash
node --experimental-vm-modules --test loadtests/tests/geodata-workloads.test.mjs
node loadtests/tests/k6-smoke.mjs
```

| Evidence | Result | Scope |
| --- | --- | --- |
| [14 harness regressions](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/tests/geodata-workloads.test.mjs) | 14 passed | Route sequence, checksums, part limits, malformed/lost responses, abort, safe diagnostics, production safeguards and exact-tag teardown |
| [Real k6 transport smoke](https://github.com/myota-platform/myota-geodata-service/blob/main/loadtests/tests/k6-smoke.mjs) | 8,463,234 bytes; two parts; 4/4 checks; 0/6 failed requests; zero fixtures remaining | Disposable localhost protocol server; no real accounts/database/S3 writes |
| Python regressions with isolated PostGIS | 90 discovered; 89 passed; one two-API-server test skipped | All migrations applied to disposable `geodata_upload_tests`; cleanup cascades parts, preserves other tags and refuses active uploads |
| [Upload cleanup tests](https://github.com/myota-platform/myota-geodata-service/blob/main/tests/test_load_test_upload_fixtures.py) | Five unit cases plus real-database cascade test passed | Exact tag/ID/status predicates and transaction/foreign-key behavior |
| Ruff | Format and lint checks passed | Service Python files, including cleanup helper/tests |

The [image workflow](https://github.com/myota-platform/myota-geodata-service/blob/main/.github/workflows/build-and-publish.yml)
gates publication on both harness and Python/database regressions. Pull requests
run those checks without publishing an image. Real SeaweedFS transfer/restart
qualification and all representative non-production workload runs remain
separate roadmap gates; a mocked protocol server cannot establish capacity.

## Update and recovery

Update the checkout containing the script before rerunning your existing
runner. Preserve local `runtest.sh` edits; it is not replaced by this fix.

```bash
git pull --ff-only
./runtest.sh
```

Confirm the log says `uploads=resumable-v1`. Production thresholds are evaluated
after the workload rather than aborting early; final failures are still failures.
Explicit environment/host acknowledgements, workload caps and separately
enabled cleanup remain mandatory; see the
[full load-test runbook](geodata-load-test-and-query-evidence.md).

If teardown fails, stop new workloads and use the same target/credentials:

```bash
MYOTA_LOAD_TEST_PROFILE=cleanup-only \
MYOTA_LOAD_TEST_RUN_ID=lt-20261007151608-large-upload \
k6 run loadtests/geodata-workloads.js
```

That example is the failed run in the supplied log. Its existing cleanup
already reported zero imports/entities/objects removed and `cleaned=true`;
the log does not show retained accepted imports. Do not delete unrelated data.
For a future interrupted resumable upload, the harness logs its upload ID.
Authenticate as its owner and abort that exact session through
`DELETE /v1/geodata/import-uploads/{uploadId}`, then retry exact-tag cleanup.
Cleanup refuses active transfers and protected QSO/award links; it never
bypasses those safeguards or automatically deletes unrelated sessions.
