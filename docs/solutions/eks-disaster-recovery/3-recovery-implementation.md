# Recovery Implementation

> **Phase:** 3 Execute
> **Evidence standard:** The vault, KMS key, IAM roles, plan, selection, and S3
> logical-backup path are applied and inspected. Full clean-room reconstruction
> and logical restore remain acceptance targets.

## Abstract

This phase builds the three protection layers decided in [recovery
architecture and decisions](2-recovery-architecture-and-decisions.md) and
leaves each one inspectable. It covers what is created, who owns it, and what
must be observable before validation begins.

It does not prove recovery. A completed backup job is evidence that a
mechanism ran, not that a service can be restored. Proof belongs to [restore
validation](4-restore-validation-and-results.md).

## Entry criteria

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Recovery objectives approved | Every protected datastore has an owner-approved RPO and RTO | Requirements register | Service owner |
| Layer decisions recorded | EBS selection, logical-backup, and reconstruction decisions are accepted with their rejected alternatives | [Phase 2 decisions](2-recovery-architecture-and-decisions.md) | Platform recovery lead |
| Key and account boundaries agreed | Customer-managed KMS key, vault account, and access separation are approved | Security decision record | Security owner |
| Source of truth exists | Infrastructure and Kubernetes desired state are held in Terraform and Argo CD | Repository revisions | Platform recovery lead |

## Roles

| Role | Supplies | Signs off |
|---|---|---|
| Platform recovery lead | Terraform for vault, key, role, plan, selection, and alarms | Layer 1 and layer 3 implementation |
| Data recovery lead | Database-native backup jobs and their validation queries | Layer 2 implementation per engine |
| Security owner | KMS grants, IRSA scoping, bucket policy, and retention separation | Permission and retention boundaries |
| Service owner | Protected namespaces, volumes, and datastore inventory | Selection scope for their service |

## Domain index

| Domain | Required output | Detail | Owner |
|---|---|---|---|
| Layer 1: volume recovery points | Tag-scoped EBS selection, encrypted vault, retention rules, and orphan reconciliation | [Three-tier backup implementation](3-1-three-tier-backup-implementation.md) | Platform recovery lead |
| Layer 2: logical artifacts | Engine-specific export jobs writing versioned, prefix-scoped S3 objects with completion markers | [Three-tier backup implementation](3-1-three-tier-backup-implementation.md) | Data recovery lead |
| Layer 3: deterministic reconstruction | Terraform and Argo CD sources sufficient to rebuild the platform without manual memory | [Three-tier backup implementation](3-1-three-tier-backup-implementation.md) | Platform recovery lead |
| Guardrails | Failure, freshness, count, orphan, partial-job, and budget signals | [Three-tier backup implementation](3-1-three-tier-backup-implementation.md) | Platform recovery lead |

## Known pitfalls

| Pitfall | Why it happens | How to avoid it |
|---|---|---|
| Selection scope silently widens or narrows | Tag conditions are edited without reconciling against the live PVC inventory | Compare each scheduled run against expected volumes and alert on count drift in both directions |
| Backup writers can delete their own history | One role is reused for write and lifecycle operations | Separate writer permissions from retention permissions and enforce retention outside the protected cluster |
| Absent backups go unnoticed | Monitoring alerts on failures only | Alert on missing completion markers and unexpected recovery points, not just explicit failures |
| Clustered database volumes are treated as independent | Layer 1 protects volumes, not database consistency | Pair volume protection with an engine-native logical artifact for every clustered datastore |
| Reconstruction depends on undocumented manual steps | Bootstrap actions are performed once and never recorded | Record every manual dependency not represented in Terraform or Argo CD as an explicit implementation gap |

## Exit criteria

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Layer 1 applied | A scheduled run produces the expected recovery-point count with no unintended resource types selected | AWS Backup job history | Platform recovery lead |
| Layer 2 applied | Each engine produces an artifact and a completion marker in its own S3 prefix | Object listing and marker check | Data recovery lead |
| Layer 3 represented | Infrastructure and Kubernetes desired state reconcile from source with no undeclared manual step | Terraform plan and Argo CD sync state | Platform recovery lead |
| Retention enforced externally | Backup writers cannot shorten retention or delete recovery points | Rendered IAM and bucket policies | Security owner |
| Guardrails active | Freshness, failure, count, orphan, partial-job, and budget signals have subscribed destinations | Alarm configuration and test notification | Platform recovery lead |
| Implementation gaps declared | Every unexercised mechanism is recorded as a gap rather than presented as capability | Implementation status table | Platform recovery lead |

## Continue

- Next phase: [Restore validation and results](4-restore-validation-and-results.md)
- Domain detail: [Three-tier backup implementation](3-1-three-tier-backup-implementation.md)

## References

- [AWS Backup developer guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [Amazon EKS best practices](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
