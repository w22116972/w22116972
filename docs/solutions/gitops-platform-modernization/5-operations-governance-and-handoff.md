# Operations, Governance, and Handoff

## Abstract

This phase defines the operating model once the normal change interface
becomes a reviewed desired-state change rather than a cluster command. It
covers drift response, break-glass, promotion and rollback governance,
monitoring, reconstruction, onboarding, and the acceptance test for
independent operation.

## Operating model

GitOps changes the normal operational interface from a cluster command to a
reviewed desired-state change. It does not remove operational ownership.

| Activity | Platform team | Application team | Security/cloud team | On-call operator |
|---|---|---|---|---|
| AppProject and cluster destination policy | Accountable | Consulted | Consulted | Informed |
| Platform controller lifecycle | Accountable/responsible | Informed | Consulted for IAM | Operates runbook |
| Product chart, image, and test evidence | Consulted | Accountable/responsible | Consulted | Observes rollout |
| IAM and workload identity | Consulted | Consulted | Accountable/responsible | Escalates failures |
| Secret value lifecycle | Consulted | Consulted | Accountable/responsible | Uses approved recovery path |
| Stateful sync and membership check | Accountable | Consulted | Informed | Responsible during window |
| Break-glass change | Accountable for reconciliation | Consulted | Consulted as needed | Responsible for stabilization |
| Disaster recovery | Shared | Shared for application/data validation | Shared for cloud/credentials | Executes coordinated runbook |

## Drift response runbook

1. Identify the Application, desired revision, resource, field difference, and
   current sync/health state.
2. Determine whether drift came from a person, an old writer, a mutating
   webhook, a controller-generated object, or API-server defaulting.
3. Assess user impact before syncing. `OutOfSync` alone is not authorization to
   change a live resource.
4. If Git is correct and automation is approved, reconcile and run the
   workload-specific effective-state checks.
5. If live state is the correct emergency fix, pause automation and encode the
   change in Git through an expedited review.
6. If another controller owns the field, remove it from Argo ownership or apply
   a narrow, justified diff rule—never ignore an entire resource casually.
7. Verify the prior writer is disabled, restore the intended sync policy, and
   record cause, resolution, and preventive action.

## Break-glass procedure

Break glass requires an incident owner and expiry time.

```text
declare incident and affected Application
        |
        v
pause automated sync for that Application
        |
        v
apply the smallest reversible stabilization
        |
        v
verify service behavior and capture the effective diff
        |
        v
commit the fix or revert the cluster change
        |
        v
confirm Argo diff is empty or explained
        |
        v
restore approved automation and close the incident record
```

For stateful services, the procedure also checks quorum/membership, PVCs, and
data integrity. For gateway or load-balancer changes, it checks external
addresses, routes, targets, and deletion side effects before re-enabling prune.

## Promotion and rollback governance

Promotion is by pinned version or revision. The promoting pull request includes
the source-environment result, rendered diff, known environment differences,
rollback revision, and observation owner. High-risk changes use a maintenance
window and manual sync even when ordinary changes are automated.

Rollback policy is change-aware:

- stateless configuration normally uses Git revert and automated reconciliation;
- controller upgrades use the prior pinned chart plus controller-specific
  webhook, identity, and cloud-side checks;
- Argo self-upgrades remain manual and retain a bootstrap recovery path;
- stateful changes stop after the first affected member and use the database
  recovery/membership runbook; and
- data loss or corruption invokes the separately tested data-protection plan,
  not a blind manifest rollback.

## Monitoring and alerts

The operations view needs both control-plane and workload signals:

| Signal | Why it matters | Initial response |
|---|---|---|
| Application `OutOfSync` beyond reconciliation window | Desired and live state differ | Classify writer/defaulting/controller cause |
| Application `Degraded`, `Missing`, or long `Progressing` | Sync success may hide runtime failure | Inspect health message, rollout, events, image and dependencies |
| Repeated sync failure | Desired state may be invalid or admission is rejecting it | Stop retries, review exact failed resource and wave |
| Repository or render failure | Argo cannot calculate desired state | Check repository credentials, revision, chart availability, and render output |
| Orphaned resource | Ownership inventory or prune scope may be incomplete | Determine legitimate owner before deletion |
| Unexpected manual sync-policy change | Automation may have been paused and forgotten | Correlate with incident/change record |
| Controller/webhook unavailable | Reconciliation or admission can fail across domains | Use controller-specific readiness and dependency runbook |
| PVC, quorum, or database health change | Stateful availability/data may be at risk | Stop further member syncs and invoke stateful runbook |

Alerts route to the lifecycle owner identified by the Application and resource
class. A platform team should not silently absorb failures caused by an
application-owned image, and an application team should not be expected to
repair the GitOps control plane.

## Recovery and reconstruction

Cluster recovery is an ordered reconstruction:

1. Terraform restores the network, EKS, nodes, IAM, workload identity, and
   managed add-ons.
2. The bootstrap procedure installs Argo CD and establishes repository access.
3. The root Application recreates projects and child Application definitions.
4. Platform domains reconcile in dependency order.
5. The external credential system restores Secret material.
6. Stateful data is restored through datastore-specific recovery procedures.
7. Operators verify gateways, controllers, observability, storage, database
   membership, and application behavior.

Git is the source for declared cluster state, not a substitute for data backup.
The complementary [Amazon EKS disaster-recovery case study](../eks-disaster-recovery/README.md)
defines the volume and database-native protection layers. Recovery remains
incomplete until reconstruction and data restore have been exercised together.

## Runbook set

The handoff package includes:

- ownership and escalation matrix;
- new Application/AppProject onboarding checklist;
- existing-resource adoption checklist;
- drift diagnosis and reconciliation;
- failed sync and failed rollout diagnosis;
- break-glass and return-to-Git procedure;
- controller upgrade and rollback;
- stateful one-member-at-a-time sync;
- repository credential recovery;
- root/ApplicationSet recovery; and
- cluster reconstruction plus data-restore coordination.

Runbooks use symbolic examples and observable acceptance conditions. They do
not embed customer identifiers, credentials, or raw production output.

## Onboarding checklist

Before a team receives sync access, it must demonstrate that it can:

1. identify the authoritative repository and Application owner;
2. explain the difference between source, sync, health, rollout, and effective
   state;
3. render and review its desired state locally or in CI;
4. identify secrets, IAM, storage, and controller-generated boundaries;
5. use the correct environment promotion path;
6. diagnose a simulated `OutOfSync` and `Progressing` Application;
7. pause and restore automation through the break-glass procedure;
8. execute or locate the workload-specific rollback; and
9. return every emergency change to reviewed source.

## Independent-operation acceptance

Handoff is complete when the customer/platform operators, without delivery-team
intervention, can:

- onboard a low-risk Application within a restricted AppProject;
- explain and resolve a controlled drift exercise;
- promote and roll back a pinned revision;
- recover a deleted low-risk resource through reconciliation;
- diagnose a synchronized but unhealthy workload;
- perform break glass and reconcile back to Git;
- operate the manual stateful sync gate; and
- recover the Argo root and repository connection from the documented
  bootstrap path.

Attendance at a knowledge-transfer session is not sufficient evidence. The
operators perform the exercises, capture results, and own any follow-up gaps.

## Open governance gates

- Establish a defensible baseline before claiming lead-time or failure-rate
  improvement.
- Complete an end-to-end cluster reconstruction and datastore restore exercise.
- Resolve the current auxiliary image availability exception and add provenance
  or mirroring controls.
- Review the one automated monitoring Application that intentionally omits
  prune and document its orphan-management policy.
- Reassess each manual stateful Application after backup, membership, and
  rollback exercises; manual may remain the correct steady state.
- Migrate product application pipelines only through a separately scoped
  ownership decision.
