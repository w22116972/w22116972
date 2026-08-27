# Recovery Runbook

> **Status:** Exercised (path A) · Unproven (paths B and C)
> **Last exercised:** 2026-08-19, path A only, restore without data readback
> **Owner:** Platform recovery lead

## Abstract

This runbook restores an EKS-hosted service into an isolated target, validates
it, and returns traffic through an explicit approval gate. It is intentionally
provider-neutral where a command would expose environment identifiers.

Never restore directly over the affected service when the failure could be
logical corruption. Preserve the original environment for comparison and
forensics.

The order of this procedure is fixed. It branches only once, at step 4, on a
classification discovered during investigation, and the branches re-converge
at the traffic gate.

Path A has been exercised and timed, but without data readback. Paths B and C
have never completed a drill. They are written procedures, not proven recovery
capability, and must not be reported as one.

## Roles

| Role | Responsibility |
|---|---|
| Incident commander | Declares recovery, owns timeline and business decisions |
| Service owner | Selects valid recovery point and approves data and application checks |
| Platform recovery lead | Reconstructs EKS, IAM, storage, networking, and GitOps |
| Data recovery lead | Restores the datastore and records validation evidence |
| Security owner | Approves KMS, secret, break-glass, and cross-account access |
| Change approver | Authorizes traffic switch and rollback |
| Scribe | Records timestamps, commands, evidence, exceptions, RPO, and RTO |

One person may fill several roles in a small exercise, but each decision must
still be attributed.

## Entry criteria

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Recovery point identified | A recovery point exists that is older than the suspected corruption | Vault or object listing | Service owner |
| Recovery access available | KMS decrypt, restore-role assumption, and secret retrieval succeed from the recovery environment | Access check output | Security owner |
| Isolation available | A target account, cluster, or namespace exists that cannot reach production | Target inventory | Platform recovery lead |
| Engine compatibility confirmed | Database, operating-system, CSI, and Kubernetes versions are compatible with the artifact | Version comparison | Data recovery lead |
| Capacity available | Storage is sufficient for the restore plus validation work | Capacity check | Platform recovery lead |

Abort if the recovery point, ownership, encryption key, engine compatibility,
or isolation cannot be verified.

## Procedure

1. Record the incident declaration time and the affected business services.
   - **Verify:** The scribe holds a UTC declaration timestamp and a named
     incident commander.
2. Stop automated writes if continued writes increase corruption or divergence.
   - **Verify:** Writer processes are confirmed stopped or fenced, and the stop
     time is recorded.
3. Preserve logs, events, audit records, and the affected storage.
   - **Verify:** The original volume or dataset is retained untouched and is
     excluded from any restore target.
4. Classify the failure and select exactly one path.

   | Failure class | Path |
   |---|---|
   | Deleted Kubernetes object with healthy data | Reconcile from Argo CD or Git; this runbook does not apply |
   | Lost or unusable EBS volume | Path A |
   | Logical corruption or tenant deletion | Path B |
   | Namespace loss | Recreate namespace and dependencies, then Path A or B |
   | Cluster loss | Path C, then Path A or B for data |
   | Account compromise | Independent-account recovery; current implementation gap |

   - **Verify:** Exactly one path is selected and the classification is recorded
     with the evidence that produced it.
5. Identify the last known-good transaction or application marker, then choose
   the newest recovery point older than the suspected corruption.
   - **Verify:** The chosen recovery point predates the corruption marker, and
     the expected RPO is recorded before the restore begins.
6. Prepare the isolated target.
   - Use symbolic recovery names that cannot collide with production resources.
   - Block external traffic and unapproved egress.
   - Disable scheduled jobs, notifications, webhooks, and integrations that
     could create real side effects.
   - Capture the target inventory before making changes.
   - **Verify:** The target is confirmed unreachable from production, and a
     pre-change inventory is stored.
7. Execute the selected path below.
   - **Verify:** The path's own final step has passed.
8. Request the traffic gate.
   - **Verify:** Every row of the traffic-gate list is satisfied and the change
     approver has recorded promote, continue, or abort.

### Path A: restore an EBS volume

1. Select the recovery point by source volume, creation time, status, and KMS
   key.
   - **Verify:** Recovery-point metadata matches the intended source volume.
2. Start an AWS Backup restore into the recovery environment.
   - **Verify:** The restore job reaches a completed state.
3. Record the new volume identifier and elapsed time.
   - **Verify:** Both values are in the evidence record.
4. Attach the volume to a dedicated recovery node, or mount it through a
   scratch Pod with read-only access first.
   - **Verify:** The mount succeeds read-only before any writable mount.
5. Validate filesystem type, mount state, expected directories, file sizes,
   and checksums.
   - **Verify:** Checksums or sizes match the expected manifest.
6. Start a compatible datastore in isolation. Do not attach an independently
   restored set of clustered database volumes and assume consistency.
   - **Verify:** The datastore starts and reports a consistent state.
7. Run datastore integrity and application-level queries.
   - **Verify:** Integrity checks pass and the service owner accepts the query
     results.

If filesystem or database validation fails, detach the volume, retain it for
diagnosis, and try an earlier recovery point. Do not modify the original
source.

### Path B: restore a logical artifact

1. Require a completion marker, expected manifest, checksum, object version,
   and approved recovery date.
   - **Verify:** All five are present; a missing completion marker disqualifies
     the artifact.
2. Download through a read role separate from the backup writer.
   - **Verify:** The download used the read role, not the writer role.
3. Restore into an empty, isolated database.
   - **Verify:** The target database was empty before the restore began.
4. Apply engine-specific validation.

   | Engine | Minimum validation |
   |---|---|
   | PostgreSQL | Roles, extensions, schemas, indexes, constraints, sequences, critical row counts, application queries |
   | MongoDB | Database and collection counts, indexes, sampled documents, application queries |
   | OpenLDAP | Base DN, entry count, schema, bind, lookup, and group membership |
   | Oracle | Import log, schemas, invalid objects, row counts, constraints, application queries |
   | Neo4j | Database inventory, indexes, constraints, node and relationship counts, sampled Cypher, store-copy convergence |

   - **Verify:** Every row for the restored engine passes and results are
     recorded.
5. Compare the restored marker with the incident time and calculate data loss.
   - **Verify:** Calculated loss is within the approved RPO, or a signed
     exception exists.
6. Reconcile any newer valid transactions through an approved replay or manual
   process.
   - **Verify:** The reconciliation set is enumerated and the service owner
     accepts it.
7. Obtain service-owner validation before promotion.
   - **Verify:** Written acceptance is in the evidence record.

### Path C: reconstruct the platform

Apply dependencies in this order:

```text
AWS account access and KMS
  -> networking and DNS prerequisites
  -> EKS control plane and nodes
  -> EBS CSI and core add-ons
  -> ingress or Gateway API and certificates
  -> Argo CD bootstrap
  -> platform controllers and CRDs
  -> managed Secrets
  -> datastores and restored data
  -> applications
  -> observability and backup schedules
  -> validation and traffic approval
```

1. Pin the source revisions and container artifacts used for recovery.
   - **Verify:** Terraform, Argo CD, Helm, and image references are recorded as
     immutable identifiers.
2. Run Terraform plans and require review before apply.
   - **Verify:** A reviewer approved the plan and the applied result matches it.
3. Bootstrap Argo CD with only the root application applied manually.
   - **Verify:** No application other than the root was applied by hand.
4. Wait for platform prerequisites before reconciling dependent applications.
   - **Verify:** CRDs and platform controllers report healthy before dependent
     applications sync.
5. Restore Secrets from the approved source, never from copied plaintext notes.
   - **Verify:** Secret material came from the approved authority.
6. Restore data through path A or path B.
   - **Verify:** The selected path's final step has passed.
7. Validate cluster health, storage, DNS, certificates, routes, workloads,
   observability, and backup schedules.
   - **Verify:** Each area reports healthy and new backups are being produced.
8. Record every manual dependency not represented in source.
   - **Verify:** The list is filed as an implementation gap with an owner.

### Traffic gate

The incident commander may request traffic approval only when:

- backup origin, timestamp, and integrity are recorded;
- database and application checks pass;
- expected data loss is within the approved RPO or an exception is signed;
- latency, error, saturation, and dependency signals are healthy;
- authentication and authorization checks pass;
- write paths and idempotency have been tested with synthetic data;
- monitoring and new backups are active in the target; and
- rollback remains possible.

The change approver records one of three decisions: promote, continue isolated
validation, or abort.

## Abort conditions

| Condition | Why it is disqualifying | Action |
|---|---|---|
| Recovery point, ownership, encryption key, engine compatibility, or isolation cannot be verified | The restore would be unverifiable or could reach production | Abort before step 6 |
| No failure class matches the observation | The procedure has no validated path for the situation | Escalate; do not default to the most familiar path |
| Filesystem or database validation fails | The recovery point is unusable | Retain the volume for diagnosis and retry with an earlier point |
| Logical artifact has no completion marker | The artifact may be truncated | Reject the artifact and select an earlier one |
| Calculated data loss exceeds the approved RPO without a signed exception | The business has not accepted the loss | Stop at the traffic gate |
| Integrity, error, or latency limits breach after promotion | The recovered service is not correct | Roll back |
| Writes diverge between environments | Automatic selection would destroy unique data | Stop and perform an explicit reconciliation decision |
| An undeclared dependency appears | The recovered state is incomplete | Roll back and record the gap |

## Rollback

Promotion establishes the boundary beyond which rollback becomes a data
decision rather than a traffic decision.

For promotion:

1. Establish a final write boundary on the old service.
   - **Verify:** The old service rejects new writes.
2. Replay or reconcile approved changes made after the recovery point.
   - **Verify:** The reconciliation set is enumerated and accepted.
3. Take a final pre-cutover validation snapshot or logical artifact.
   - **Verify:** The artifact exists and is restorable.
4. Shift a small test percentage or an internal hostname first.
   - **Verify:** Synthetic and internal journeys pass on the new path.
5. Observe application and business signals, then expand traffic gradually.
   - **Verify:** Signals stay inside approved thresholds at each increment.
6. Keep the previous environment isolated and recoverable until the rollback
   window closes.
   - **Verify:** The previous environment is intact and reachable by the
     recovery roles.

To roll back, return traffic to the prior service only if doing so will not
overwrite newer valid data. If both states contain unique writes, stop and
perform an explicit data-reconciliation decision rather than choosing one
automatically.

The last reversible point is the final write boundary in step 1. After that,
reversal requires reconciliation, not routing.

## Evidence record

Every drill or incident must record:

| Field | Required value |
|---|---|
| Incident or exercise | Sanitized identifier and scenario |
| Declaration time | UTC timestamp |
| Failure class and path | Classification, selected path, and the evidence behind it |
| Recovery point | Type, creation time, status, checksum or immutable reference |
| Target | Account, region, cluster, namespace, and isolation controls |
| Source revisions | Terraform, Argo CD, Helm, and application versions |
| Start and stop | Recovery start, data-ready, app-ready, and traffic-ready timestamps |
| Validation | Commands or queries, expected result, actual result, and owner |
| RPO and RTO | Calculation and approved exception if the target was missed |
| Decision | Promote, abort, or retry, with approver |
| Cleanup | Temporary resources and retained forensic evidence |
| Follow-up | Owner, due date, and permanent corrective action |

Store the record in the incident system or controlled recovery repository. Do
not publish raw logs, credentials, customer data, internal resource
identifiers, or unredacted screenshots.

## Compliance

A procedure that is never exercised decays silently while still looking
authoritative. The `Status` and `Last exercised` fields above are only
trustworthy if these checks run.

| Check | Mechanism | Cadence |
|---|---|---|
| Path A still works | Restore one small EBS volume and validate the filesystem read-only | Monthly |
| Path B still works | Restore one representative logical database and run application checks | Quarterly |
| Path C still works | Reconstruct an isolated EKS environment from Terraform and Argo CD | Semiannual |
| The procedure still matches the system | Repeat the affected drill after retention, KMS, IAM, storage, database topology, or platform-bootstrap changes | On change |
| Deviations feed back | Every deviation recorded in the evidence record is reviewed and either becomes a runbook edit or a recorded exception | After each run |
| Status is honest | `Status` and `Last exercised` are reviewed against exercise records; an expired drill downgrades the path to unproven | Quarterly |

The full cadence and its ownership are defined in [operations and
handoff](5-operations-and-handoff.md).

## Notes

Paths B and C have never been drilled. Account compromise has no path at all
and is recorded as an implementation gap in [recovery
implementation](3-recovery-implementation.md).

## References

- [AWS Backup developer guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [Operations and handoff](5-operations-and-handoff.md)
