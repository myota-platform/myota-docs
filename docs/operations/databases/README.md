# Database migrations and recovery

The three baseline rebuilds are complete. This index links to the source
inventory and the dated cutover record:

- [Baseline migration inventory](baseline-migration-inventory.md) — authoritative
  schema owners, synchronized mirrors, and migration-runner design.
- [Database rebuild evidence](database-rebuild-evidence-2026-10-13.md) — exact
  archive checksums, isolated restore rehearsals, live cutovers, validation, and
  cleanup.

Activity and Geodata own their schemas; Deploy owns migration orchestration and
keeps deployment copies synchronized with the service sources. Core is shared
by Identity and Programme. See the inventory for the ownership map and the
closed issue checklists.
