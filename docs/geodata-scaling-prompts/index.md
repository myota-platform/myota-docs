# ChatGPT prompts: geodata horizontal scaling

Use each page as a separate ChatGPT coding task. The pages follow the open
items in the [geodata horizontal-scaling roadmap](../geodata-horizontal-scaling-roadmap.md).
Phase 1 is already closed; Phase 0, 2, 3, 4, and 5 still have open qualification
or implementation gates.

## Pages

1. [Phase 0 — baseline workload and evidence](phase-0-baseline.md)
2. [Phase 2 — upload recovery across restarts](phase-2-upload-recovery.md)
3. [Phase 3 — bounded workers and failure recovery](phase-3-worker-isolation.md)
4. [Phase 4 — infrastructure and scaling constraints](phase-4-infrastructure.md)
5. [Phase 5 — staged rollout and operational proof](phase-5-rollout.md)

Run them in order where practical. Phase 0 supplies the measurements Phase 4
needs; Phases 2 and 3 establish the recovery behavior Phase 5 must prove. A
later page may discover a dependency on an earlier gate; record that evidence
and continue independent work without marking the dependent gate complete.
