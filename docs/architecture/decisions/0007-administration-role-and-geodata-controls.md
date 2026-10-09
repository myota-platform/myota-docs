# ADR-0007: Least-privilege administration and protected geodata maintenance

## Status

Accepted

## Decision

The administration web exposes account editing, multiple active role
assignments, and custom role definitions through the identity service. Built-in
roles provide a small hierarchy: Global administrator, Identity administrator,
GIS administrator, Geodata approver, Programme administrator, Activity
administrator, and Auditor. Custom roles may select only allowlisted
administrative permissions; wildcard access remains reserved for the global
administrator. The last global administrator cannot be removed.

The geodata review queue applies catalogue filters in the geodata API before
deterministic pagination. It supports programme (including unassigned
entities), entity type, continent, country, region/subdivision,
province, city/municipality, and multi-select lifecycle status. The map renders
the current result page and remains independently pannable/zoomable; selecting
an item centres the map without changing the queue. Global or GIS
administrators may convert point, line, and polygon entity geometries; the old
geometry and reason remain in history. Reviewers may also change shared category
codes and display names for platform-wide or programme-assigned entities; the
old value and reason remain in review history. Only rejected entities may be permanently
deleted, and deletion removes the entity, related conflation candidates, and
its audit record.

## Rationale

Administrative responsibilities should be composable without granting every
operator global access. Rejected records are not valid programme references,
so controlled deletion is safe for them; approved and retired records remain
protected to preserve historical activity references.
