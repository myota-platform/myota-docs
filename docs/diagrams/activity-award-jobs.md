# Activity and award jobs

```mermaid
flowchart LR
  Client[Participant or administrator]
  Activation[Activation resource]
  Ingestion[QSO ingestion job]
  Worker[Bounded activity worker]
  Aggregate[(Relational aggregates)]
  Evaluation[Award evaluation or recalculation]
  Render[Certificate render job]
  Artifact[(SeaweedFS certificate artifact)]
  Client --> Activation
  Client --> Ingestion
  Ingestion --> Worker
  Activation --> Worker
  Worker --> Aggregate
  Aggregate --> Evaluation
  Evaluation --> Worker
  Client --> Render
  Render --> Worker
  Worker --> Artifact
```

All job state is exposed through the activity service on port 8004. Workers
retry transient failures with bounded backoff, and every award evaluation uses
the programme-owned rule version attached to the job.
