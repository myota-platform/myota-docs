# Database migrations and recovery

- [Baseline migration inventory and rebuild status](baseline-migration-inventory.md)
  — authoritative owners, synchronized mirrors, runner checks, archive evidence,
  live boundaries, and the remaining gates for the three database rebuilds.

Deploy owns the combined migration runner and deployment wiring. Activity and
Geodata own their schema baselines; Deploy and Platform keep synchronized copies.
The inventory links to each source repository and tracks current status.
