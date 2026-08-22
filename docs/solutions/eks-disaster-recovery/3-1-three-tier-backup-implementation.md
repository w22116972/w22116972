# Three-Tier Backup Implementation

## Implementation status

This chapter is a sanitized implementation record. It distinguishes the
production reference design from the smaller non-production deployment used to
exercise it.

| Component | Implemented state | Evidence boundary |
|---|---|---|
| AWS Backup vault, KMS key, IAM role, and plan | Applied | Current plan and selection inspected through the AWS API |
| Tier 1 EBS selection | Applied with ANDed cluster and namespace conditions | Latest scheduled run produced 45 EBS-only recovery points, all completed |
| Tier 1 retention | Daily with seven-day retention in non-production | Active plan has one `son-daily` rule |
| Tier 2 S3, lifecycle, bucket policy, and IRSA | Applied | Infrastructure and rendered policies inspected |
| Tier 2 database jobs | Built and backup mechanisms exercised, then disabled in non-production | Artifact creation was observed before disablement |
| Tier 3 Terraform and Argo CD | Active source of truth | Full clean-room reconstruction not yet drilled |
| Restore | EBS volume creation exercised | Data and application validation not completed |

## Layer 1: EBS recovery points

Terraform owns the vault, customer-managed KMS key, AWS Backup role, plan,
selection, lifecycle, notifications, alarms, and orphan reconciliation.

The active non-production plan is intentionally small:

| Rule | Schedule | Retention | Effective state |
|---|---|---:|---|
| Son | Daily | 7 days | Enabled |
| Father | Weekly | 28 days | Disabled by environment toggle |
| Grandfather | Monthly | 120 days | Disabled by environment toggle |

The complete GFS shape remains in code so production can enable the longer
tiers through reviewed variables. Reducing cadence was rejected because EBS
snapshots are incremental: the age of the oldest retained point, not simply the
number of scheduled runs, controls how long overwritten blocks remain billable.

### Effective selection verification

The final selection combines:

- an EBS-only ARN pattern;
- a required EKS cluster ownership tag; and
- namespace exclusions for data classified as reproducible.

Read-only verification on 2026-08-19 found:

| Check | Result |
|---|---|
| Active plan rules | One daily rule, seven-day deletion lifecycle |
| Selection API | EBS resource pattern plus `StringEquals` and `StringNotEquals` conditions |
| `ListOfTags` | Empty |
| Latest scheduled inventory | 45 recovery points |
| Resource types | 45 EBS, 0 EC2, 0 EKS |
| Status | 45 `COMPLETED` |

This closes the earlier union-selection defect. It does not prove that every
business-critical PVC is present; that requires reconciling the 45 selected
volumes with the current Kubernetes data inventory and ownership catalog.

### Retention and monitoring

The vault publishes job events and CloudWatch metrics. Guardrails include:

- backup job failure;
- absence of a completed job within 48 hours;
- partial composite recovery points, retained as a tripwire if EKS-type backups
  are reintroduced;
- recovery-point count above an independently set steady-state threshold;
- unexpected snapshots not owned by AWS Backup; and
- actual and forecast snapshot spend.

The failure metric is sparse: AWS publishes a value when a failure occurs, not
a continuous stream of zeroes. Metric math uses `FILL(metric, 0)` so an old
failure does not leave the alarm permanently latched and unable to signal a new
failure.

The orphan detector is read-only. Automated deletion is intentionally excluded;
an operator reviews resource tags, ownership, protection in other systems, and
rollback evidence before cleanup.

## Layer 2: database-native artifacts

Logical backups are rendered from one Helm chart into engine-specific CronJobs
and service accounts. Terraform owns the S3 and IAM side; Argo CD owns only the
in-cluster jobs.

```text
database
   |
   v
engine-native dump container
   |
   v
size-limited scratch space or direct S3 stream
   |
   v
IRSA uploader --> s3://example-backup-bucket/<cadence>/<engine>/<namespace>/<date>/
                                                          |
                                                          v
                                                   _COMPLETE.json
```

`_COMPLETE.json` is written last. Dump objects without the marker prove that an
upload started, not that the backup completed.

One daily export can be copied server-side into weekly and monthly prefixes on
the relevant dates. Prefix-first key layout makes independent lifecycle rules
possible without teaching a CronJob how to delete history:

```text
daily/postgres/namespace-a/2026-08-19/database.dump
daily/postgres/namespace-a/2026-08-19/_COMPLETE.json
weekly/postgres/namespace-a/2026-08-17/database.dump
monthly/postgres/namespace-a/2026-08-01/database.dump
```

Reference logical retention is 14 daily, 35 weekly, and 180 monthly days.
Incomplete multipart uploads expire after seven days.

### Engine contracts

| Engine | Backup method | Consistency and validation notes | Restore unit |
|---|---|---|---|
| PostgreSQL | Globals export plus custom-format dump per database | MVCC snapshot; list archive and validate roles, extensions, schemas, counts, and constraints | Database |
| MongoDB | Gzipped archive with oplog capture where supported | Restore into scratch instance and compare database and collection counts | Database or selected namespace |
| OpenLDAP | Full-tree LDIF export | Fail if the export is empty; validate base DN, entry count, and authentication | Directory tree |
| Oracle | Full export using a client compatible with the deployed server | Require non-empty dump and successful log; validate schemas and row counts | Database or schema |
| Neo4j | Topology-aware native backup directly to S3 | Supply all current server endpoints; validate every expected database and artifact | Database |
| Redis | No logical backup for reconstructible cache | Rebuild from source; do not promote cache to business data accidentally | Not applicable |

Backup jobs first verified that each engine could produce an artifact. Examples
included a multi-gigabyte PostgreSQL dump completed in minutes, a MongoDB archive,
small LDAP and Oracle exports, and hundreds of Neo4j database artifacts during a
measured topology-aware run. These are backup-generation results only; none is
used as evidence of a successful logical restore.

### Pod and scheduling controls

- `concurrencyPolicy: Forbid` prevents overlapping runs.
- `activeDeadlineSeconds` bounds a stuck job and is increased for the
  topology-wide Neo4j run.
- The engine tool runs before the uploader through init-container ordering.
- Scratch data uses a size-limited ephemeral volume and is removed with the Pod.
- CPU and memory requests are engine-specific; a JVM walking hundreds of
  databases is not assigned the same budget as a small LDAP export.
- Jobs retry failures conservatively and retain enough history for diagnosis.

Neo4j writes directly to S3 and uses scratch space only for the database being
processed. The scratch limit is based on the largest observed database, not the
sum of the cluster.

### Workload identity and bucket policy

Each engine has a separate IRSA role. Trust is bound to exact service-account
subjects rather than `system:serviceaccount:*`, preventing a newly created
namespace from assuming a backup role merely by choosing the same service
account name.

The writer's mutating permissions allow only:

```text
s3:PutObject
s3:AbortMultipartUpload
```

for the engine's daily, weekly, and monthly prefixes. It omits
`s3:DeleteObject`, and the bucket policy explicitly denies deletion by backup
writers. Read and list permissions are limited to the same engine prefixes so a
job can verify uploads and continue an incremental backup chain. Versioning,
SSE-S3 encryption, public-access blocking, and lifecycle are managed centrally.

Database credentials are read from Kubernetes Secrets. Where an application
did not already use a Secret, a bootstrap step copied the runtime value into a
dedicated Secret without writing it to Git. This was expedient but does not
solve cluster-loss recovery; a managed external secret source remains required.

## Layer 3: deterministic reconstruction

| Platform layer | Source | Recovery action |
|---|---|---|
| Networking, EKS, IAM, node groups, add-ons, backup services | Terraform | Initialize clean state and apply reviewed modules |
| Controllers, CRDs, Gateway API, policies, backup jobs | Argo CD app-of-apps | Bootstrap Argo CD and apply the root application |
| Application workloads | Reviewed Helm and Git sources | Reconcile through Argo CD or the delivery pipeline |
| Secrets | Managed external source required | Rehydrate before dependent workloads start |

Git stores desired intent and history. It does not capture uncommitted live
changes, mutable external resources, container images that have been deleted,
or credentials that exist only in the cluster. A reconstruction drill must
inventory those dependencies rather than treating `terraform apply` as proof of
service recovery.

## Implementation sequence

The safe rollout order was:

1. inventory workloads, PVCs, storage classes, database topology, and owners;
2. define RPO, RTO, retention, and exclusions;
3. create KMS, vault, IAM, S3, lifecycle, and monitoring through Terraform;
4. deploy logical-backup service accounts and jobs through Argo CD;
5. exercise each backup mechanism and verify completion artifacts;
6. run a new EBS selection before retiring the previous mechanism;
7. create an isolated volume through AWS Backup;
8. measure the effective resource set and correct selection semantics;
9. remove legacy snapshots only after exact ownership checks; and
10. keep the incomplete data and platform restore drills visible as release
    gates.

The legacy cleanup removed hundreds of stale snapshots after checking that each
target belonged to the intended environment, was not AWS Backup-managed, and
did not belong to another cluster. Age was an assertion, not the deletion
selector. That distinction prevented a broad age-based cleanup from deleting a
different environment's active protection.
