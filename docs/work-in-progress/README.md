# Work in progress

This index points to active delivery and evidence gates. Completed implementation
does not close independent rollout or acceptance work. Broader unstarted work
is listed in [To do](../to-do/README.md).

## Cross-repository architecture review

Phases 1–4 are implemented and documented in [REVIEW.md](../../REVIEW.md) and
the [phase evidence record](../architecture/evidence/review-phases-1-4-2026-10-11.md).
They restore source-to-mirror parity, establish the shared logger source,
extract the Geodata route map, and retire unused API aliases. The separate
Phase 0 duplicate-YAML-key check remains open.

## Active implementation and evidence

- [Geodata scale qualification](../geodata/horizontal-scaling-roadmap.md) —
  Phase 5 remains open for broader capacity, independent-row consistency,
  node/storage failure, canary, and rollback evidence.
- [Structured logging acceptance](../observability/logging-implementation.md) —
  release-state reconciliation, authenticated Grafana navigation,
  request-to-worker correlation, Loki field privacy, and Compose parity remain
  open in the evidence record.
- [Observability live status](../observability/overview.md#live-k3s-verification-and-known-gap)
  — Identity and Programme metrics coverage requires remediation and current
  cluster verification.
