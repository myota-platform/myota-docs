# Cross-repository review — Phases 1–4 evidence

**Completed:** 2026-10-11  
**Team:** Codex and Volker  
**Source of scope:** [REVIEW.md](../../../REVIEW.md)  
**Change policy:** direct commits to each repository's main; Deploy and
Platform runtime files are mirrors, not independent domain owners.

## Phase 1 — owner-to-mirror parity

**Prompt used**

~~~text
Implement Phase 1 of the cross-repository review. Read REVIEW.md and the
repository map first. Treat the five service repositories as owners of their
runtime modules and migrations, Contracts as the owner of API/event contracts,
Deploy as the image and Helm owner, and Platform as an integration mirror.
Map every synchronized path or tree that the deployment/bootstrap repositories
consume. Preserve database ownership and migration order. Add a deterministic
CI audit that compares file bytes and directory contents, then build and deploy
the resulting images. Record exact source revisions, image digests, and
readiness evidence. Do not edit mirrors as independent authorities.
~~~

**Outcome:** Contracts maps 44 individual source files and 92 file mirrors,
plus five exact tree mirrors: the Platform event-schema tree and the Activity
and Geodata migration trees in Deploy and Platform. The map also covers the
canonical OpenAPI/event registry and Operations migration copies. Deploy is
the image source; Platform contains integration copies. The audit compares
file bytes and whole-tree file sets.

**Evidence**

- [Contracts source-mirror map](https://github.com/myota-platform/myota-contracts/blob/main/contracts/runtime-mirror-map.json)
  and [audit script](https://github.com/myota-platform/myota-contracts/blob/main/scripts/audit_runtime_mirrors.py).
- Contracts CI run [38145106871](https://github.com/myota-platform/myota-contracts/actions/runs/38145106871) passed on `16f800936ea81edc0f0672216eaf135b2c00d519`: 44 owner files, 92 file mirrors, and five exact tree mirrors.
- Owner quality/image workflows passed at the commits and run IDs in the final verification matrix below. Platform CI passed run [38144778620](https://github.com/myota-platform/myota-platform/actions/runs/38144778620).
- Deploy quality and image workflows passed on `f7d2e65edd5248d70376cf99ccc98c836b7f95e3` (runs [38144677934](https://github.com/myota-platform/myota-deploy/actions/runs/38144677934) and [38144677329](https://github.com/myota-platform/myota-deploy/actions/runs/38144677329)). Fleet reports Ready=True at that commit. Helm release `myota` is deployed at revision 246, and every listed Deployment has ready replicas equal to its desired count.

## Phase 2 — shared runtime helpers

**Prompt used**

~~~text
Implement Phase 2 of the cross-repository review. Read the review and Phase 1
source map before changing helpers. Keep service authorization, persistence,
and error policy service-owned. Select the lowest-complexity distribution
approach that preserves independent service builds. Establish one canonical
logging module and a repeatable exact-content sync/check for all required
build roots. Inventory common.py differences and document which adapters remain
service-local. Run focused service quality checks and update source links.
~~~

**Decision:** use generated source synchronization for myota_logging.py. Its
canonical copy is myota-platform/runtime/myota_logging.py; the sync script
distributes it to five service roots and the Deploy/Platform runtime trees. It
remains a source module so each service image builds independently. The
common.py modules remain service-local because they combine different HTTP,
persistence, upload, metrics, and handler adapters. Shared-looking copies are
not assigned an unreviewed cross-service release dependency.

**Evidence:** the source/mirror audit checks the canonical logger against all
seven copies. The five owner service quality/image workflows, Deploy Python
quality/image workflow, Platform CI, and Contracts CI pass at the revisions
listed in the final status. Runtime behavior and data ownership are unchanged.

## Phase 3 — service hotspot and boundaries

**Prompt used**

~~~text
Implement Phase 3 of the cross-repository review. Read REVIEW.md, ADR-0005,
the repository map, and the Geodata and Activity designs first. Keep Geodata as
the owner of spatial entities/imports and Activity as the owner of activation,
QSO, and award execution data. Extract the Geodata route registrations into
cohesive route groups without changing handlers, authorization, transactions,
outbox writes, idempotency, recovery, or API behavior. Keep existing workers
and persistence modules in their owners. Document the Programme/Activity award
boundary and avoid creating another service or database without evidence.
~~~

**Decision:** geodata_routes.py now contains the catalogue, import, and job
route maps, and is synchronized into Deploy and Platform. Existing import,
relational-state, and worker modules retain their ownership. No new service or
database was justified. geodata.py remains a large handler module; a deeper
handler split is deferred because its class/state coupling has not shown a
safe, independently testable boundary in this change. This is a recorded
maintainability residual, not a claim that the file is small.

**Evidence:** Geodata quality and image workflows pass; Contracts route
inventory contains no missing or unregistered handlers. The Deploy and
Platform mirrors match the owner source.

## Phase 4 — API alias retirement and architecture record

**Prompt used**

~~~text
Implement Phase 4 of the cross-repository review. Read Phases 1–3 evidence and
the REST API consolidation record. Inspect Contracts, every owning service,
both first-party web clients, Deploy, Platform, and the shared .github links.
Remove compatibility-only aliases that have no first-party callers in this
pre-live system. Preserve explicit audited commands and job-confirmation
boundaries. Update canonical OpenAPI, client coverage, route inventory,
mirrors, diagrams, indexes, and implementation history. Do not add runtime
traffic counters or user migration steps for routes no client uses.
~~~

**Outcome:** unused aliases were removed from Identity, Programme, Activity,
Geodata, Operations, OpenAPI, and first-party clients. Audited lifecycle
commands, job resources, and the Geodata/Activity deletion workflow remain.
Service deprecation response headers, alias registries, and alias-specific
metrics were removed. Current web source searches contain no references to
the removed aliases.

**Evidence:** Contract CI reports 132 API operations and 170 route
registrations, with no missing routes or duplicate operation IDs. Identity,
Programme, Activity, Geodata, Operations, Deploy, and Platform quality/build
workflows pass at their final source revisions. The route inventory is
regenerated and checked in.

## Final decision and remaining work

Phases 1–4 are complete for their recorded scope. The separate Phase 0
duplicate-key and handler/mirror gate is also complete; see the
[Phase 0 closure record](review-phase-0-2026-10-11.md). Contracts CI now
self-tests duplicate-key rejection, checks the canonical OpenAPI and mirrors,
and regenerates a 131-operation inventory against 170 service route
registrations with no missing routes or duplicate operation IDs. The existing
Geodata handler hotspot is documented for future measured refactoring; it does
not block the current API, mirror, or ownership decisions.


## Final verification matrix

| Owner/repository | Main revision | Focused CI evidence |
| --- | --- | --- |
| Identity | `a503fbeffdc4d8a3ae9aeeee3607d81e5ea891fc` | [Quality](https://github.com/myota-platform/myota-identity-service/actions/runs/38144653398), [image](https://github.com/myota-platform/myota-identity-service/actions/runs/38144652432) passed |
| Programme | `ae65e0e9b80173e710e1265da804c3bd9cd3cbdc` | [Quality](https://github.com/myota-platform/myota-programme-service/actions/runs/38144657940), [image](https://github.com/myota-platform/myota-programme-service/actions/runs/38144657278) passed |
| Activity | `3095559cac35f3c0c9a8d021c387e298ac0cf95b` | [Quality](https://github.com/myota-platform/myota-activity-service/actions/runs/38144662340), [image](https://github.com/myota-platform/myota-activity-service/actions/runs/38144661896) passed |
| Geodata | `ad28632723765463fa683ac175cf430045c769f0` | [Quality](https://github.com/myota-platform/myota-geodata-service/actions/runs/38144667559), [image](https://github.com/myota-platform/myota-geodata-service/actions/runs/38144667133) passed |
| Operations | `91b5c2ffdb7bc82419e556d3dc9604624d218972` | [Quality](https://github.com/myota-platform/myota-operations-service/actions/runs/38144671669) passed |
| Deploy | `f7d2e65edd5248d70376cf99ccc98c836b7f95e3` | [Quality](https://github.com/myota-platform/myota-deploy/actions/runs/38144677934), [image](https://github.com/myota-platform/myota-deploy/actions/runs/38144677329) passed |
| Platform | `22c9a6523b39de12734900c25722e998935af4d6` | [CI](https://github.com/myota-platform/myota-platform/actions/runs/38144778620) passed |
| Contracts | `16f800936ea81edc0f0672216eaf135b2c00d519` | [Contract freeze and mirror audit](https://github.com/myota-platform/myota-contracts/actions/runs/38145106871) passed |

The current K3s observation used the explicit host kubeconfig. Fleet's
`myota-deploy` GitRepo reported Ready=True at
`f7d2e65edd5248d70376cf99ccc98c836b7f95e3`. Helm reported release
`myota`, namespace `myota`, revision 246, status `deployed`.
All application Deployments were ready at their requested replica counts.

The API and Operations images observed in the ready Deployments were:

| Deployment | Image digest |
| --- | --- |
| Identity | `sha256:aaf698323ffa791f1ea27cec257a88bb5f5183bab7ca247fb96cad26097d9cd2` |
| Programme | `sha256:a1d5bd214248c82f41524286a41c82aacb778d36d0f75899421cd60ecc3ec7e2` |
| Activity/API and workers | `sha256:557d2a3ad82ecde994cec9793da56ad3cf830e50aaf986a14cfab81f67ccf1b7` |
| Geodata/API and workers | `sha256:7599c09acecf9cbffd8d4f147af04ea2dc2ff95886464d1e08c047ef1daedd58` |
| Operations | `sha256:c4736458e9fc754d80cd5cf9022421fb91483e032fe701104622189feeb86e8e` |
| Shared Deploy runtime | `sha256:b1f89ddc7b0530df4d51427104cd87bd08686457fbb2c80a09bdbe816c5161ac` |

These are deployment observations, not a claim that every mirror change
produced a new image digest. The source CI runs above validate the source
revisions; the Deploy commit and Helm/Fleet state identify the installed
deployment configuration.
