# Profile deployment closeout — 14 October 2026

## Scope and source ownership

This closes Deployment [#12](https://github.com/myota-platform/myota/issues/12)
and the profile [workstream #55](https://github.com/myota-platform/myota/issues/55)
for isolated Testing. The monorepo owns `apps/identity`, `contracts`,
`web/participant`, and `deploy`. Former Platform mirrors now correspond to the
active `deploy/services` source copies; legacy production rollout remains the
separate monorepo transition gate, not a profile acceptance claim.

The profile implementation merged in PRs #76–#79. This closeout adds the
remaining recovery/rotation documentation and direct service-to-storage evidence.
No production configuration, user, credential, or object was changed.

## Verified gates

| Gate | Evidence |
|---|---|
| Profile implementation mirrors | `cmp` of `apps/identity/identity.py` and `deploy/services/identity.py` from `origin/testing` passed; `profile_images.py` SHA-256 matched `7e3399700603eb3f141e6d0da0a403f534d029832146cb388d7cb49d6a8af4b7`; `identity_maintenance.py` matched `ec051d7606426461c3068b57f7e530dfb855bc02f0f25e7cf52c085585d1f10f`. |
| Shared runtime boundary | Deploy common is independently maintained. Its raw upload limit/binary handling and secret/bucket wiring are verified by focused deployment tests; it is not represented as an identical Identity common mirror. |
| Compose/Helm storage checks | `PYTHONPATH=services /root/myota-platform/.venv-phase5/bin/python -m unittest tests.test_profile_image_storage -v` from `myota/deploy`: six tests passed. |
| Runtime credentials | Live Identity and maintenance deployments refer to `myota-test-profile-image-s3-auth`; SeaweedFS mounts `myota-test-seaweedfs-s3-config`. No credential value was printed or persisted in evidence. |
| Live workload availability | All 24 Deployments and three StatefulSets had their desired replicas ready; ten PVCs were bound. |
| Isolated API/storage qualification | Job `myota-profile-qualification-20261014b` completed at **21:57:40 UTC**, using deployed Identity image `sha256:7cbecb4d64db98300312237e5d776d8f8ae5ab3126a6cbe8fada58e1f77ddfdf`. |
| Cleanup and preservation | Profile bucket inventory was empty before and after the test. The synthetic account existed only in Job memory. Its object prefix was cleaned in `finally`; Job and ConfigMap deletion succeeded and neither resource remained. |
| Recovery, capacity, rotation | The [profile operations page](../../../domain/identity/profile.md) documents consistent database/SeaweedFS recovery points, isolated restore validation, secret-pair rotation/rollback, retention and capacity limits. |

## Qualification assertions

The Job started an ephemeral localhost `IdentityHandler` server using the
published deployed image. It had profile-bucket credentials from Secret
references but **no Core database URL**, and asserted that its account store
was non-durable. A synthetic active account and token remained in process
memory. The script SHA-256 was
`2d5f6be4ea0c817fdd4b89f6d9a894154d60f4641722c48b9bbb5e3176ac0bbc`.

The HTTP/storage checks verified:

- Website and Markdown biography updates returned their accepted values.
- PNG upload produced a WebP image of exactly 512 × 512 and at most 256 KiB.
- Authenticated image reads returned `Cache-Control: private, no-store`.
- Unauthenticated API image reads were denied with HTTP 403.
- Replacement removed the prior image object; explicit deletion removed the
  current image, and subsequent API reads returned HTTP 404.
- The profile storage identity could not list the unrelated `myota-adif`
  bucket or the production-profile bucket (HTTP 403).
- Anonymous listing of the Test profile bucket was denied (HTTP 403).
- Pre/post inventory equality compared every profile object key, ETag and
  size. Zero pre-existing objects means this is cleanup evidence, not a
  qualification against a populated profile corpus. Unrelated buckets were
  never written; access attempts were denied.

The test used Kubernetes Job/ConfigMap resources only in `myota-test` and deleted
both afterward. No NATS command, SMTP send, live account change or database
write occurred. Expected HTTP errors produced bounded request-error logs.

## Remaining independent limits

- No off-node backup or full profile restore was performed. The issue requires
  documented recovery behavior; this record does not claim a restore test.
- Participant DOM tests use mocked API responses. The deployed-image HTTP test
  verifies the real API-to-storage path without modifying an existing user.
- Helm revision 330 remained `pending-upgrade` during inspection, and Fleet
  displayed `WaitApplied`. Resource readiness is not proof of Fleet
  reconciliation. This remains tracked under monorepo transition #53.
- Production uses the former Deploy source until its separate cutover. The
  profile feature was not promoted or deployed there by this closeout.
