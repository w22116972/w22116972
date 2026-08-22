# Restore Validation and Results

## Evidence standard

Recovery evidence progresses through six gates. A result at one gate must not
be described as passing a later gate.

| Gate | Evidence | What it proves |
|---:|---|---|
| 1 | Policy and resources exist | The mechanism is configured |
| 2 | Backup job reports success | The provider completed its workflow |
| 3 | Artifact or recovery point is present and current | A candidate recovery input exists |
| 4 | Resource is restored in isolation | The restore API and permissions work |
| 5 | Data is read and validated | The restored bytes are usable and internally consistent |
| 6 | Application checks pass and traffic is approved | The service has recovered |

Only gate 6 closes a service-recovery drill. RPO and RTO are recorded only after
the selected recovery point and service-validation time are known.

## Verified results

Evidence was rechecked against committed implementation sources and read-only
AWS APIs on 2026-08-19.

| Scenario | Backup used | Target | Validation | Observed RPO | Observed RTO | Result | Gap |
|---|---|---|---|---|---|---|---|
| Scheduled EBS protection | Daily AWS Backup plan | Tagged EBS volumes | 45 recovery points, all EBS and `COMPLETED` | Not measured against an incident | Not applicable | Gate 3 passed | Reconcile selected volumes with current critical-PVC inventory |
| EBS restore mechanism | One small EBS recovery point | New isolated volume | Restore API completed and produced an encrypted volume | Not recorded | 76 seconds to volume creation | Gate 4 passed | Volume was deleted before mount, checksum, datastore, or application validation |
| PostgreSQL backup creation | Native logical export | S3 artifact | Non-empty multi-gigabyte dump and completion marker | Schedule supports 24-hour target | Backup completed in minutes | Gate 3 passed | No restore or application query |
| MongoDB backup creation | Native archive | S3 artifact | Non-empty archive and completion marker | Schedule supports 24-hour target | Backup completed in minutes | Gate 3 passed | No restore or collection validation |
| LDAP and Oracle backup creation | LDIF and database export | S3 artifacts | Non-empty artifacts and successful tool results | Schedule supports 24-hour target | Backup completed | Gate 3 passed | No authentication or application validation after restore |
| Neo4j backup path | Native topology-aware backup | Per-database S3 artifacts | Hundreds of database artifacts produced in a measured run | Schedule supports 24-hour target | Generation rate measured; restore not timed | Gate 3 partially passed | Complete inventory and restore-seed validation required |
| Platform reconstruction | Terraform and Argo CD | New EKS environment | Source and bootstrap sequence inspected | Git-history target | Not measured | Gate 1 passed | No clean-room reconstruction |

The EBS restore closes meaningful risks: AWS Backup could assume its restore
role, decrypt the recovery point with the configured KMS key, and create a new
volume. It does not prove that the source filesystem or database can start.

## Scope-correction result

The most useful outcome came from counting effective recovery points. An early
configuration intended to select 66 EBS volumes but produced 129 recovery
points: 120 EBS volumes and nine EC2 instances. `Resources` and `ListOfTags` had
formed a union.

After replacing `ListOfTags` with ANDed `Conditions` and applying namespace
exclusions, the current run produced 45 EBS recovery points and no EC2 recovery
points. All completed. This validates the selection semantics and current
resource types; it still needs a workload-to-volume completeness check whenever
the cluster inventory changes.

## Required complete restore drill

PostgreSQL is a suitable first full drill because it exercises logical recovery,
roles, extensions, schema, data, and an application query without requiring a
multi-node data topology.

### Preparation

1. Select a database with a named service owner and representative data.
2. Record the incident simulation start time, latest committed transaction or
   durable application marker, backup timestamp, engine versions, and expected
   object counts.
3. Create an isolated namespace and network policy. Do not reuse the production
   service name or credentials.
4. Confirm that the target has enough ephemeral and persistent storage and that
   the restore tool is compatible with the source version.

### Restore

1. Download the globals artifact, database dump, manifest, checksum, and
   `_COMPLETE.json` from one recovery date.
2. Verify object version, encryption access, expected size, and checksum before
   loading anything.
3. Start an empty PostgreSQL target with traffic isolated.
4. Restore required roles and globals with environment-specific ownership
   reviewed.
5. Restore the selected custom-format dump with errors treated as fatal.
6. Capture tool logs and the exact end time.

### Data validation

- run `pg_restore --list` and compare the archive inventory with expectations;
- verify required extensions, schemas, tables, indexes, constraints, sequences,
  and ownership;
- compare row counts for critical tables and sample referential-integrity
  queries;
- compare a pre-recorded business marker or checksum;
- run database consistency and application-specific invariant checks; and
- record every accepted difference.

### Application validation

1. Deploy the application version compatible with the recovery point.
2. direct only synthetic traffic to the isolated database;
3. exercise authentication, read, write, search, and one representative
   business transaction;
4. verify logs, latency, errors, and downstream side effects;
5. obtain service-owner sign-off; and
6. calculate actual RPO and RTO from recorded timestamps.

The drill passes only if the data and application checks succeed. If they fail,
keep the original service untouched, preserve the failed target for diagnosis,
select an earlier recovery point if corruption is suspected, and restart the
documented procedure.

## Neo4j recovery validation

Neo4j needs a separate drill because database placement spans servers. The
validation sequence is:

1. capture the expected database list and primary/secondary topology;
2. select one representative tenant database and validate its backup artifact;
3. restore it to one isolated server using a compatible Neo4j version;
4. validate counts, constraints, indexes, sampled node and relationship
   checksums, and representative Cypher queries;
5. add clean peers and observe store-copy and consensus convergence;
6. confirm every expected primary becomes available; and
7. test the application against the recovered routing endpoint.

Restoring all server volumes as if they were a coordinated snapshot is not an
accepted drill because independent EBS recovery points do not guarantee a
single database-consistent cluster time.

## Platform reconstruction drill

A clean-room drill should create a new recovery environment without copying
live Kubernetes objects:

1. apply networking, KMS access, IAM, EKS, nodes, and required add-ons;
2. establish storage classes, CSI drivers, DNS, certificates, and ingress;
3. bootstrap Argo CD and apply the root application;
4. rehydrate Secrets from the approved managed source;
5. restore data into isolated services;
6. reconcile applications in dependency order;
7. run platform and application acceptance tests; and
8. switch a test DNS name only after approval.

Record manual commands and resources that were missing from source. Each is a
reconstruction defect to fix before the next drill.

## Cost and operational results

- Non-production retention was reduced to a seven-day horizon by disabling the
  weekly and monthly rules, rather than reducing the daily cadence and weakening
  RPO.
- Logical retention is independent from EBS retention, allowing small,
  selectively restorable artifacts to be kept longer than expensive volume
  history.
- EBS resource selection removed the per-namespace creation cost associated
  with EKS composite backups and stopped protecting disposable node volumes.
- Hundreds of legacy snapshots were removed only after ownership and replacement
  protection were checked. The cleanup left other environments unchanged.
- Recovery-point count and budget alarms make unbounded retention visible.

These are architecture and inventory outcomes. No percentage savings claim is
made because a normalized before/after billing window with workload-change
adjustments is not available in this case study.

## Remaining release gates

- complete the PostgreSQL data and application drill;
- complete a topology-aware Neo4j restore;
- reconstruct an isolated cluster from Terraform and Argo CD;
- externalize all required Secrets;
- connect alert notifications to an owned, tested response channel;
- add and test cross-account copies for account-level failure;
- verify KMS recovery access from the independent environment;
- measure actual service RPO and RTO; and
- repeat drills on a documented cadence.

Until these gates pass, the accurate status is **backup mechanisms implemented,
recovery capability partially validated**.
