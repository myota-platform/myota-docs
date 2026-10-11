# Work in progress

This index points to active delivery and evidence gates. Completed implementation
does not close independent rollout or acceptance work. Broader unstarted work
is listed in [To do](../to-do/README.md).

## Cross-repository architecture review

Phases 0–4 are complete and documented in [REVIEW.md](../../REVIEW.md), the
[Phase 0 closure record](../architecture/evidence/review-phase-0-2026-10-11.md),
and the [Phases 1–4 evidence record](../architecture/evidence/review-phases-1-4-2026-10-11.md).
The contract cleanup definition, duplicate-key CI gate, handler check, and
canonical/root/Platform mirrors are reconciled.

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
