# Architecture and ownership

Start with [Current system state](current-state.md) for the latest verified
repository heads, deployed image digests, K3s/Fleet status, NATS topology, and
open evidence gates.

- [Platform architecture](overview.md) — service boundaries, data ownership,
  security, and cross-service flows.
- [Repository map](repository-map.md) — target monorepo ownership, transition repositories, and synchronized
  Deploy/Platform mirrors.
- [Monorepo and future-scale assessment](monorepo-and-scalability-assessment.md)
  — accepted consolidation, current implementation status, scale design, and
  language qualification criteria.
- [Architecture decisions](decisions/README.md) — durable decisions and
  rationale.
- [Identity account and role policy](decisions/0009-identity-accounts-and-role-policy.md) — account creation, role authority, registration gates, and implementation status.
- [Diagrams](diagrams/README.md) — service/data boundaries plus the separate
  source/deployment transition status.
- [Cross-repository review](../../REVIEW.md) — findings, mitigation, and
  completion evidence.
