# Architecture and Decisions

## Migration architecture

The migration uses separate paths for schema and data. Schema assessment finds
objects that convert automatically and action items that need redesign. AWS DMS
then performs a full load followed by CDC while the source remains authoritative.
The application moves only after validation and an explicit go/no-go decision.

## Target selection

Evaluate Aurora PostgreSQL and RDS for PostgreSQL against compatibility,
availability, scaling, performance, operational complexity, recovery,
extensions, and cost. A heterogeneous Oracle-to-PostgreSQL example can guide
the process, but it is not evidence that every source should use the same target.

## Validation layers

- AWS DMS validation compares source and target records and exposes mismatches.
- Independent queries compare counts, keys, nulls, ranges, aggregates, and
  referential integrity.
- Application and business owners validate critical workflows and totals.
- Performance tests compare representative query and transaction behavior.

AWS DMS validation consumes resources and has limitations; it supplements rather
than replaces business reconciliation.

## Rollback boundary

Before cutover, the source remains authoritative. During cutover, writes are
paused or tightly controlled, final CDC lag reaches the approved threshold, and
business reconciliation completes. Rollback is allowed only while the source
can be restored as authoritative without losing accepted target-only writes.
