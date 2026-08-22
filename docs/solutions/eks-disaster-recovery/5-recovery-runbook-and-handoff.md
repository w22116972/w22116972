# Recovery Runbook and Handoff

## Purpose

This runbook restores an EKS-hosted service into an isolated target, validates
it, and returns traffic through an explicit approval gate. It is intentionally
provider-neutral where a command would expose environment identifiers.

Never restore directly over the affected service when the failure could be
logical corruption. Preserve the original environment for comparison and
forensics.

## Roles

| Role | Responsibility |
|---|---|
| Incident commander | Declares recovery, owns timeline and business decisions |
| Service owner | Selects valid recovery point and approves data/application checks |
| Platform recovery lead | Reconstructs EKS, IAM, storage, networking, and GitOps |
| Data recovery lead | Restores the datastore and records validation evidence |
| Security owner | Approves KMS, secret, break-glass, and cross-account access |
| Change approver | Authorizes traffic switch and rollback |
| Scribe | Records timestamps, commands, evidence, exceptions, RPO, and RTO |

One person may fill several roles in a small exercise, but each decision must
still be attributed.

## Declare and classify

1. Record incident declaration time and affected business services.
2. Stop automated writes if continued writes increase corruption or divergence.
3. Preserve logs, events, audit records, and the affected storage.
4. Classify the failure:

| Failure class | Preferred path |
|---|---|
| Deleted Kubernetes object with healthy data | Reconcile from Argo CD or Git |
| Lost or unusable EBS volume | Layer 1 EBS restore |
| Logical corruption or tenant deletion | Layer 2 logical restore |
| Namespace loss | Recreate namespace and dependencies, then restore data |
| Cluster loss | Layer 3 reconstruction, then layer 1 or 2 data restore |
| Account compromise | Independent-account recovery path; current implementation gap |

5. Identify the last known-good transaction or application marker.
6. Choose the newest recovery point older than the suspected corruption.
7. Record expected RPO before restore begins.

## Prepare an isolated target

- Use symbolic recovery names that cannot collide with production resources.
- Block external traffic and unapproved egress.
- Confirm KMS decrypt access, restore-role assumption, and secret availability.
- Confirm compatible database, operating-system, CSI, and Kubernetes versions.
- Allocate enough storage for restore plus validation work.
- Disable scheduled jobs, notifications, webhooks, and integrations that could
  create real side effects.
- Capture the target inventory before making changes.

Abort if the recovery point, ownership, encryption key, engine compatibility,
or isolation cannot be verified.

## Path A: restore an EBS volume

1. Select the recovery point by source volume, creation time, status, and KMS
   key.
2. Start an AWS Backup restore into the recovery environment.
3. Wait for completion and record the new volume identifier and elapsed time.
4. Attach the volume to a dedicated recovery node or mount it through a scratch
   Pod with read-only access first.
5. Validate filesystem type, mount state, expected directories, file sizes, and
   checksums.
6. Start a compatible datastore in isolation. Do not attach an independently
   restored set of clustered database volumes and assume consistency.
7. Run datastore integrity and application-level queries.
8. Promote only through the traffic gate.

If filesystem or database validation fails, detach the volume, retain it for
diagnosis, and try an earlier recovery point. Do not modify the original source.

## Path B: restore a logical artifact

1. Require a completion marker, expected manifest, checksum, object version,
   and approved recovery date.
2. Download through a read role separate from the backup writer.
3. Restore into an empty, isolated database.
4. Apply engine-specific validation:

| Engine | Minimum validation |
|---|---|
| PostgreSQL | Roles, extensions, schemas, indexes, constraints, sequences, critical row counts, application queries |
| MongoDB | Database and collection counts, indexes, sampled documents, application queries |
| OpenLDAP | Base DN, entry count, schema, bind, lookup, and group membership |
| Oracle | Import log, schemas, invalid objects, row counts, constraints, application queries |
| Neo4j | Database inventory, indexes, constraints, node/relationship counts, sampled Cypher, store-copy convergence |

5. Compare the restored marker with the incident time and calculate data loss.
6. Reconcile any newer valid transactions through an approved replay or manual
   process.
7. Obtain service-owner validation before promotion.

## Path C: reconstruct the platform

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

1. Pin source revisions and container artifacts used for recovery.
2. run Terraform plans and require review before apply;
3. bootstrap Argo CD with only the root application applied manually;
4. wait for platform prerequisites before reconciling dependent applications;
5. restore Secrets from the approved source, never from copied plaintext notes;
6. restore data through path A or B;
7. validate cluster health, storage, DNS, certificates, routes, workloads,
   observability, and backup schedules; and
8. record every manual dependency not represented in source.

## Validation and traffic gate

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

## Promotion and rollback

For promotion:

1. establish a final write boundary on the old service;
2. replay or reconcile approved changes after the recovery point;
3. take a final pre-cutover validation snapshot or logical artifact;
4. shift a small test percentage or internal hostname first;
5. observe application and business signals;
6. expand traffic gradually; and
7. keep the previous environment isolated and recoverable until the rollback
   window closes.

Rollback immediately when integrity checks fail, error or latency limits are
breached, writes diverge, or an undeclared dependency appears. Return traffic to
the prior service only if doing so will not overwrite newer valid data. If both
states contain unique writes, stop and perform an explicit data-reconciliation
decision rather than choosing one automatically.

## Evidence record

Every drill or incident must record:

| Field | Required value |
|---|---|
| Incident or exercise | Sanitized identifier and scenario |
| Declaration time | UTC timestamp |
| Recovery point | Type, creation time, status, checksum or immutable reference |
| Target | Account, region, cluster, namespace, and isolation controls |
| Source revisions | Terraform, Argo CD, Helm, and application versions |
| Start and stop | Recovery start, data-ready, app-ready, and traffic-ready timestamps |
| Validation | Commands or queries, expected result, actual result, and owner |
| RPO and RTO | Calculation and approved exception if target was missed |
| Decision | Promote, abort, or retry with approver |
| Cleanup | Temporary resources and retained forensic evidence |
| Follow-up | Owner, due date, and permanent corrective action |

Store the record in the incident system or controlled recovery repository. Do
not publish raw logs, credentials, customer data, internal resource identifiers,
or unredacted screenshots.

## Exercise cadence

| Frequency | Exercise |
|---|---|
| Daily | Automated freshness, failure, completion-marker, count, orphan, and budget checks |
| Monthly | Restore one small EBS volume and perform read-only filesystem validation |
| Quarterly | Restore one representative logical database and run application checks |
| Semiannual | Reconstruct an isolated EKS environment from Terraform and Argo CD |
| Annual | Cross-account or regional scenario with security, service, and business owners |
| After material change | Repeat the affected drill after retention, KMS, IAM, storage, database topology, or platform-bootstrap changes |

## Handoff checklist

- RPO, RTO, retention, and data classifications have business owners.
- Every protected namespace and datastore has a service owner.
- The selected-volume inventory is reconciled with critical PVCs.
- Logical completion markers are monitored for absence.
- Alert destinations are subscribed, acknowledged, and tested.
- Restore and break-glass roles are documented and periodically exercised.
- KMS keys and managed Secrets are available from the recovery environment.
- Cross-account copy and vault immutability decisions are recorded.
- Container images and Git sources needed for reconstruction are retained.
- PostgreSQL, Neo4j, and full-platform drills have current evidence.
- Actual RPO and RTO trends are reviewed after every exercise.
- Runbook changes flow through review and are tested before the next incident.

The handoff is complete only when a named team owns both backup alerts and
restore exercises. Owning a schedule without owning recovery validation is not
an operating model.
