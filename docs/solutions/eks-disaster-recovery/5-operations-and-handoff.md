# Operations and Handoff

> **Phase:** 5 Operate
> **Evidence standard:** The operating model and exercise cadence are defined.
> No historical handoff scorecard is retained, so independent-operation results
> are acceptance targets.

## Abstract

This phase assigns ownership of the recovery capability, sets the cadence that
keeps it exercised, and defines when the customer team owns it outright.

The governing rule is that owning a backup schedule is not an operating model.
A named team must own both backup alerts and restore exercises before handoff
is complete.

## Entry criteria

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Protection implemented | All three layers are applied with active guardrails | [Phase 3 exit criteria](3-recovery-implementation.md) | Platform recovery lead |
| Restore exercised | At least one restore path has produced measured, validated evidence | [Restore validation](4-restore-validation-and-results.md) | Data recovery lead |
| Gaps declared | Unproven paths are recorded as open drills, not as capability | Remaining release gates | Platform recovery lead |

## Roles

| Role | Supplies | Signs off |
|---|---|---|
| Business owner | RPO, RTO, retention, and data classification per service | Acceptable data loss and recovery windows |
| Service owner | Protected namespaces, datastores, and application validation queries | Restored-data correctness for their service |
| Platform recovery lead | Vault, key, plan, reconstruction sources, and alarms | Platform reconstruction readiness |
| Data recovery lead | Engine-specific backup and restore procedures | Logical restore readiness |
| Security owner | KMS, secret, and break-glass access from the recovery environment | Recovery access boundaries |
| Incident commander | Declaration, timeline, and business decisions during an event | Recovery declaration and closure |

One person may fill several roles in a small exercise, but each decision must
still be attributed.

## Domain index

| Domain | Required output | Detail | Owner |
|---|---|---|---|
| Incident recovery procedure | An exercised, step-verifiable recovery procedure with abort and rollback rules | [Recovery runbook](5-1-recovery-runbook.md) | Incident commander |
| Exercise cadence | A schedule that keeps every recovery path currently evidenced | This document | Platform recovery lead |
| Ownership transfer | A customer team that operates alerts and drills without delivery-team intervention | This document | Platform recovery lead |

## Exercise cadence

| Frequency | Exercise |
|---|---|
| Daily | Automated freshness, failure, completion-marker, count, orphan, and budget checks |
| Monthly | Restore one small EBS volume and perform read-only filesystem validation |
| Quarterly | Restore one representative logical database and run application checks |
| Semiannual | Reconstruct an isolated EKS environment from Terraform and Argo CD |
| Annual | Cross-account or regional scenario with security, service, and business owners |
| After material change | Repeat the affected drill after retention, KMS, IAM, storage, database topology, or platform-bootstrap changes |

An exercise whose evidence has expired is treated as an unproven path, not as
a previously proven one.

## Known pitfalls

| Pitfall | Why it happens | How to avoid it |
|---|---|---|
| A schedule is mistaken for a capability | Backup jobs report success and no one distinguishes mechanism from recovery | Require current restore evidence, not job history, before declaring a path ready |
| Alerts are configured but unsubscribed | Destinations are created without testing delivery | Test every alert destination and require acknowledgement during handoff |
| Recovery access is unavailable during a real event | KMS grants, restore roles, and secrets are reachable only from the environment that was lost | Prove recovery access from an isolated environment during the semiannual drill |
| Runbook drift | Retention, topology, or bootstrap changes are made without repeating the affected drill | Bind the after-material-change row of the cadence table to the change process |
| Handoff stops at a walkthrough | Independence is assumed rather than exercised | Require the customer team to complete a drill without delivery-team intervention |

## Exit criteria

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Objectives owned | RPO, RTO, retention, and data classifications each have a named business owner | Requirements register | Business owner |
| Scope reconciled | The selected-volume inventory matches the critical PVC inventory | Reconciliation report | Platform recovery lead |
| Absence monitored | Missing completion markers raise an alert, not only failures | Alarm configuration and induced-absence test | Platform recovery lead |
| Alerting proven | Every destination is subscribed, acknowledged, and delivery-tested | Test notification record | Platform recovery lead |
| Recovery access proven | Restore and break-glass roles, KMS keys, and managed Secrets are reachable from the recovery environment | Drill record | Security owner |
| Reconstruction inputs retained | Container images and Git sources required for reconstruction are retained and reachable | Retention policy and pull test | Platform recovery lead |
| Drills current | Logical-restore and full-platform drills hold unexpired evidence | Exercise records | Data recovery lead |
| Trends reviewed | Actual RPO and RTO are reviewed after every exercise against approved objectives | Review minutes | Business owner |
| Independent operation | The customer team completes a drill and an alert response without delivery-team intervention | Exercise record | Platform recovery lead |

## Continue

- Procedure: [Recovery runbook](5-1-recovery-runbook.md)
- Preceding phase: [Restore validation and results](4-restore-validation-and-results.md)

## References

- [AWS Backup developer guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
