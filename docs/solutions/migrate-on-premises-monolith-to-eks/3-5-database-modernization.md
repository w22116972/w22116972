# Database Modernization

> **Phase:** 3 Execute · **Category:** Modernization
> **Priority topic:** 24
> **Evidence standard:** Data compatibility, expand-and-contract, backup, and
> restore patterns are retained. No completed Oracle-to-Aurora, Teradata-to-
> Redshift, or production AWS DMS migration is claimed.

## Abstract

Database modernization is not automatically database-per-service. The target
must follow domain ownership, consistency, latency, availability, security,
operational capability, licensing, and recovery requirements.

## Decision model

| Need | Candidate target | Key evidence |
|---|---|---|
| Compatible managed relational engine | Amazon RDS | Feature compatibility, Multi-AZ behavior, operations and cost |
| Cloud-native relational scale or commercial-engine exit | Amazon Aurora | Schema and SQL compatibility, failover, performance and licensing model |
| Key-value or high-scale access pattern | Amazon DynamoDB | Access patterns, partition design, consistency and cost model |
| Search, graph, cache, or analytics capability | Purpose-built managed service | Business query, freshness, failure and exit requirements |
| Temporary shared relational store | Existing database with private schemas or wrapper service | Explicit owner and decomposition exit criteria |

## Modernization sequence

1. Map business domains, data owners, tables, stored logic, integrations,
   consistency requirements, recovery targets, and regulatory classifications.
2. Separate engine replatforming from domain decomposition; do not combine both
   unless the rollback and reconciliation design can absorb the risk.
3. Assess engine features and schema conversion before choosing AWS SCT, AWS
   DMS, native tools, or application-level migration.
4. Introduce a database wrapper, private schema, API, or event boundary before
   allowing a service to own its data independently.
5. Use full load plus change data capture where supported, then reconcile
   counts, checksums, domain invariants, lag, and rejected records.
6. Switch readers and writers in bounded steps; preserve the old path until the
   approved rollback window closes.
7. Remove shared-table access and obsolete stored logic only after consumers are
   proven absent.

## Exit criteria

- target engine functionality and performance are proven with representative
  queries and concurrency;
- encryption, keys, credentials, audit, residency, backup, restore, RTO, and RPO
  are approved;
- CDC lag and exception queues stay within cutover thresholds;
- old and new data reconcile by domain invariants, not only row counts;
- rollback accounts for writes accepted after cutover;
- cost includes migration tooling, data transfer, licenses, I/O, backup,
  support, and parallel-run periods.

## References

- [Database decomposition on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/database-decomposition/introduction.html)
- [Database Migration and Modernization](../database-migration-modernization/README.md)
