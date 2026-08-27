# Implementation

## Abstract

This phase sets out the delivery increments from assessment through
conversion, migration, validation, and cutover, using isolated migration
connectivity and reviewed Infrastructure as Code. It covers the security and
connectivity controls and the change discipline that keeps the source
authoritative until the go decision.

## Delivery phases

1. **Assess:** inventory schema, data, dependencies, performance, operations,
   security, and recovery requirements.
2. **Mobilize:** provision isolated migration connectivity, target database,
   secrets, monitoring, and DMS resources through reviewed Infrastructure as
   Code.
3. **Convert:** run schema assessment, remediate action items, deploy converted
   schema, and test application compatibility.
4. **Migrate:** execute full load, begin CDC, monitor errors and lag, and
   resolve validation differences.
5. **Rehearse:** perform production-like migration timing, reconciliation,
   cutover, rollback, and communication exercises.
6. **Cut over:** enforce go/no-go criteria, stop or control writes, drain CDC,
   validate, release traffic, and observe.
7. **Operate:** retain the source through the rollback window, complete
   acceptance, and decommission only through a separately approved change.

## Security and connectivity

Migration endpoints use private connectivity where feasible, scoped security
groups, encrypted connections, secrets management, and least-privilege service
roles. Logs and validation output are reviewed for sensitive data before broad
access or retention.

## Change discipline

Schema conversion, data migration, application compatibility, and production
cutover are separate gates. A successful DMS task does not prove application
correctness, acceptable performance, recovery, or operational readiness.

The current portfolio contains this implementation plan, not execution evidence.
