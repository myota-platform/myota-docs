# Documentation index

Use the [current system state](architecture/current-state.md) for the latest
verified runtime and repository snapshot. This index routes by topic; it does
not duplicate status details from that page.

## Start with a question

| If you need to… | Start here |
|---|---|
| Understand the live architecture, data ownership, or repository boundaries | [Architecture and ownership](architecture/README.md) |
| Review the proposed monorepo direction, future-scale design, or language options | [Monorepo and future-scale assessment](architecture/monorepo-and-scalability-assessment.md) |
| Find a product or API behavior | [Domain and API index](domain/README.md) |
| Run or recover the deployed system | [Operations index](operations/README.md) |
| Inspect the isolated K3s test deployment, initial seed, and image-update status | [Test deployment](operations/test-deployment.md) |
| Understand logs, metrics, traces, dashboards, or alerts | [Observability index](observability/README.md) |
| Work on spatial data and import workflows | [Geodata index](geodata/README.md) |
| Find project purpose, policy, or security controls | [Governance](governance/README.md), [platform policies](platform/README.md), and [security](security/README.md) |
| Set up development or quality checks | [Development guide](development/README.md) |
| See active work, accepted future work, or its priority | [Work in progress](work-in-progress/README.md), [To do](to-do/README.md), and [prioritized backlog](to-do/prioritized-backlog.md) |
| Find dated evidence or past changes | [History](history/README.md), plus the evidence indexes under each subject |

## Canonical document locations

- `architecture/` — current state, repository ownership, decisions, diagrams,
  cross-repository review evidence, and monorepo transition evidence.
- `domain/` — API plans and product behavior for identity, programmes,
  administration, and awards.
- `geodata/` — Geodata operations, scaling, imports, and qualification records.
- `operations/` — deployment, databases, messaging, recovery, and storage.
- `observability/` — logging, metrics, traces, Grafana, and monitoring.
- `governance/`, `platform/`, and `security/` — project direction and
  cross-cutting policy.
- `development/` — contributor workflows and quality gates.
- `work-in-progress/`, `to-do/`, and `history/` — delivery status and
  chronology.

## Which page is authoritative?

- The [current-state page](architecture/current-state.md) is the rolling
  snapshot of verified system status.
- A roadmap owns the scope, tasks, and checkboxes for its workstream.
- A dated evidence page records the result and limits of a particular check.
  Historical evidence is retained as a snapshot, even when later state changes.
- The [repository map](architecture/repository-map.md) identifies source owners
  and synchronized copies.

Older `docs/*.md`, `docs/adr/`, and `docs/diagrams/` paths remain as short
compatibility pages for external links. Use the canonical subject indexes above
for current navigation.
