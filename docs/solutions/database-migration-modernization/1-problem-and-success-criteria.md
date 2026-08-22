# Problem and Success Criteria

## Customer problem

A legacy relational database can constrain application modernization through
specialized operations, aging infrastructure, licensing, scaling limits, or
tightly coupled schema behavior. Moving data alone does not resolve these risks;
schema objects, SQL behavior, integrations, recovery, security, and operating
ownership must also migrate.

## Discovery questions

- Which engines, versions, sizes, growth rates, extensions, stored procedures,
  jobs, links, and unsupported data types exist?
- Which applications, reports, batch jobs, users, and external systems depend on
  the database?
- What are the latency, throughput, availability, recovery, retention,
  residency, encryption, and cutover requirements?
- Can source logging support CDC, and for how long can changes be retained?
- What validation evidence will the business owner accept?

## Success criteria

| Criterion | Required evidence |
|---|---|
| Compatibility | Assessment report and closure of blocking conversion actions |
| Correctness | Reconciled counts, keys, checksums/samples, constraints, and business totals |
| Replication | Full load complete and CDC lag within the approved cutover threshold |
| Performance | Critical queries and business flows meet agreed production-like objectives |
| Resilience | Backup restore, Multi-AZ behavior, and recovery procedures are exercised |
| Security | Identity, network, encryption, logging, retention, and secrets pass review |
| Cutover | Rehearsed timeline, owners, communications, go/no-go, and stop criteria |
| Rollback | Source preservation and reverse actions cover the approved rollback window |
| Operations | Customer operators accept monitoring, patching, backup, capacity, and escalation |

Templates without executed evidence do not satisfy these criteria.
