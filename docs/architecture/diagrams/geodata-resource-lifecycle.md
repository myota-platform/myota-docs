# Geodata resource lifecycle

~~~mermaid
flowchart LR
  Client[Admin or integration client]
  Import[POST geodata imports]
  Stage[Durable preprocessing work]
  Review[Candidate validation and review]
  Entity[Entity collection]
  Activity[Activity owned impact and cascade]
  Job[Geodata deletion job and confirmation]
  Client --> Import
  Import --> Stage
  Stage --> Review
  Review --> Entity
  Client --> Job
  Job --> Activity
  Activity --> Entity
  Entity -->|GET extent-filtered collection| Client
~~~

The route map is maintained in Geodata's geodata_routes.py; import,
catalogue, and job handlers remain in the Geodata service. Activity continues
to own its impact calculation, cascade, and award recalculation.
