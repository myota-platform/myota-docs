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

The geodata review queue sends the current map bounding box to the geodata API,
which applies the spatial filter before pagination. Global or GIS
administrators may convert point and polygon entity geometries; the old
geometry and reason remain in history. Only rejected entities may be permanently
deleted, and deletion removes the entity, related conflation candidates, and
its audit record.

## Rationale

Administrative responsibilities should be composable without granting every
operator global access. Rejected records are not valid programme references,
so controlled deletion is safe for them; approved and retired records remain
protected to preserve historical activity references.
