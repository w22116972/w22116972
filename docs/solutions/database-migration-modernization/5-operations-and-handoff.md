# Operations and Handoff

## Abstract

This phase divides steady-state ownership between database, application, and
security owners, defines the operational controls that keep the target
healthy, and states what must be true before handoff is accepted.

## Steady-state ownership

The database owner manages engine lifecycle, parameter changes, capacity,
performance, connections, backups, restores, replication, and maintenance. The
application owner manages SQL compatibility, connection pools, transactions,
and release behavior. Security owns identity, network, encryption, audit, and
exceptions. Service operations own objectives, paging, escalation, and customer
communications.

## Operational controls

- Monitor availability, connections, storage, CPU, memory pressure, I/O,
  replication, locks, slow queries, and backup status.
- Test restore procedures and record achieved recovery time and recovery point.
- Review query plans and capacity trends before thresholds become incidents.
- Patch and upgrade through compatibility tests and a reversible maintenance
  plan.
- Keep migration-specific DMS resources only as long as rollback, audit, or
  follow-up replication requires them.

## Exit criteria

Provide architecture decisions, inventories, schema action items, IaC ownership,
database and migration settings, validation evidence, dashboards, alerts,
runbooks, backup and restore results, security controls, cost model, cutover and
rollback records, and open risks.

Customer operators must demonstrate incident diagnosis, credential rotation,
backup restore, capacity response, and application escalation. Source shutdown
is a separate approval after the rollback window and data-retention obligations
have closed.
