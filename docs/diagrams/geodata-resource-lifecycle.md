# Geodata resource lifecycle

```mermaid
flowchart LR
  Client[Admin or integration client]
  Import[POST geodata imports]
  Stage[Preprocessing queue]
  Review[Candidate validation and review]
  Entity[Entity collection]
  Activity[Activity service impact and cascade]
  Job[Deletion job]
  Client --> Import
  Import --> Stage
  Stage --> Review
  Review --> Entity
  Client --> Job
  Job --> Activity
  Activity --> Entity
  Entity -->|GET bbox-filtered collection| Client
```

The legacy action routes are aliases into the same resource handlers and are
marked with deprecation headers. Tile delivery remains separate from the
filtered entity collection.
