# Recovery Requirements and Failure Model

## Business problem

An Amazon EKS platform can lose service without losing the cluster. A user can
delete one tenant, a migration can corrupt a schema, a persistent volume can
fail, or an operator can remove the entire environment. Each incident needs a
different recovery unit and sequence.

The recovery design therefore begins with business data and failure modes, not
with a backup product. Its goals are to:

- bound acceptable data loss and downtime;
- restore the smallest safe unit;
- keep recovery data outside the failure domain it protects;
- reconstruct reviewed platform intent instead of replaying unknown drift; and
- make restore evidence, ownership, and remaining risk visible.

The targets below are engineering acceptance targets for the reference design,
not a contractual SLA. A customer business-impact analysis must approve them
before production use.

## Workload and data classes

| Class | Examples | Source of truth | Recovery priority |
|---|---|---|---:|
| Business databases | PostgreSQL, MongoDB, Neo4j, LDAP, Oracle | Database state plus logical backups | 1 |
| File-backed application data | Uploaded or generated files on EBS | Persistent volume and application metadata | 1 |
| Platform configuration | VPC, EKS, IAM, controllers, CRDs, workloads | Terraform, Argo CD, and Helm in Git | 2 |
| Credentials | Database passwords, certificates, API credentials | Managed secret store; this is incomplete in the observed implementation | 2 |
| Cache and transient data | Redis cache, queues that can be replayed, mail spool | Upstream system or application rebuild | 3 |
| Observability data | Logs and metrics with separate retention | Observability backend | 3 |

Recovery priority is dependency order, not business value in isolation. A
database can be priority 1 while still waiting for networking, IAM, encryption
keys, and storage drivers to exist.

## Recovery objectives

| Workload or data class | RPO target | RTO target | Backup method | Restore method | Retention model | Latest evidence |
|---|---:|---:|---|---|---|---|
| EBS-backed business state | 24 hours | 4 hours to an isolated, readable volume | Daily AWS Backup recovery point | Create volume, mount in isolated workload, validate, then promote | Seven daily points in non-production; longer tiers configurable | Volume creation completed in 76 seconds; readback not performed |
| One database or tenant | 24 hours | 2 hours | Database-native export to S3 | Restore selected artifact into a scratch database, validate, then migrate or promote | Prefix-based daily, weekly, and monthly lifecycle | Artifact creation verified; restore not yet drilled |
| Complete data service | 24 hours | 8 hours | EBS plus logical backup | Recreate service, restore data, validate topology and application queries | Matches workload criticality | Not yet demonstrated |
| Infrastructure and platform | Approved Git history | 8 hours to a reconciled base platform | Terraform and Argo CD source | Apply infrastructure, bootstrap Argo CD, reconcile applications | Git retention and repository protection | Sources exist; isolated reconstruction not timed |
| Kubernetes Secrets | Undefined until externalized | Undefined | Managed secret source required | Rehydrate through External Secrets or encrypted Git workflow | Secret-store policy | Uncovered gap |
| Cache or transient data | No preservation target | 1 hour after dependencies return | None by design | Rebuild or replay | Not applicable | Classification only |

RPO is calculated from the timestamp of the last validated recovery point to the
incident time. RTO starts at incident declaration and ends only when validation
passes and traffic is approved. A backup job duration or volume creation time is
not service RTO.

## Production and non-production policies

Non-production data is reproducible and does not justify the same history as
customer data. The observed environment therefore uses one daily EBS rule with
seven-day retention and keeps the logical jobs disabled after exercising their
mechanisms.

A production policy should enable only the tiers justified by the impact
analysis. The reference configuration supports:

| Tier | Typical cadence | Reference retention | Purpose |
|---|---|---:|---|
| Son | Daily | 7 days for EBS; 14 days for logical artifacts | Recent operational recovery |
| Father | Weekly | 28 days for EBS; 35 days for logical artifacts | Older corruption discovery |
| Grandfather | Monthly | 120 days for EBS; 180 days for logical artifacts | Long-tail business recovery |

These are defaults, not universal requirements. Retention must follow legal,
contractual, and business needs, and must be verified against actual storage
growth and restore time.

## Failure model

| Failure | Covered by | Recovery unit | Limitation or escalation |
|---|---|---|---|
| Kubernetes object deletion | Argo CD reconciliation or Git reconstruction | Object, application, or platform | Live-only objects and Secrets are not fully reconstructible |
| PVC or EBS volume loss | AWS Backup | Whole volume | Recovery point is crash-consistent, not database-consistent |
| Bad migration or logical corruption | Database-native S3 artifact | Database or tenant | Restore drill and clean-point selection remain required |
| Namespace loss | Git reconstruction plus logical or EBS restore | Namespace and its data | Ownership and dependency order must be recorded |
| Cluster loss | Terraform, Argo CD, EBS, and S3 | New cluster | Full reconstruction has not been timed end to end |
| Region loss | Git can rebuild elsewhere; logical artifacts are portable | New regional environment | No verified cross-region data copy in the implementation |
| Account compromise or malicious deletion | Independent copy and immutable retention | New account | Cross-account copy and vault lock are not enabled |
| Credential loss | External managed secret source | Secret set | Current implementation has a blocking gap |
| Silent schedule failure | Freshness and completion-marker alerts | Backup process | Notification ownership must be connected and exercised |
| Retention failure and cost growth | Lifecycle policies, recovery-point counts, orphan reconciliation, budgets | Recovery inventory | Thresholds must be independent of the policy inputs |

## Threat and dependency boundaries

The design assumes an operator or compromised in-cluster job may have access to
the protected workload. It therefore keeps retention enforcement outside the
cluster and denies deletion to backup writers. It does not yet withstand a
principal that can delete the vault, KMS key, and S3 bucket in the same AWS
account.

Recovery also depends on services that are easy to overlook:

- the customer-managed KMS key for recovery points and the S3 encryption
  configuration for logical artifacts;
- IAM roles and EKS workload identity;
- EBS CSI and snapshot support;
- DNS, certificates, ingress or Gateway API, and external dependencies;
- compatible database binaries; and
- Git and artifact registries reachable from the recovery environment.

Each dependency must either be reconstructed from source or explicitly included
in the recovery plan.

## Acceptance criteria

Recovery readiness requires all of the following:

- the effective AWS Backup resource set matches the intended EBS volumes;
- every expected scheduled run produces a completion signal within the RPO;
- at least one important datastore is restored into isolation and read back;
- checksums, schemas, row or object counts, constraints, and application queries
  pass;
- the platform is reconstructed from a clean environment without copying live
  cluster state;
- Secrets and encryption-key access are available from managed sources;
- traffic returns only after an accountable human approval;
- actual RPO and RTO are recorded from drill timestamps;
- abort and rollback paths are exercised; and
- cross-account or equivalent immutability controls match the threat model.

The current implementation meets backup-creation and basic EBS restore-mechanism
criteria. It does not yet meet the data, application, full-platform, credential,
or account-compromise criteria.
