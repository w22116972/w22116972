# Solution Context and Requirements

## Abstract

This document is the solution contract for migrating self-managed Kubernetes
to Amazon EKS. It defines the enterprise reference scenario, the target
requirements the design must satisfy, and the non-goals it excludes, so that
every later phase can trace a decision back to a requirement recorded here.

## Evidence standard

All quantities and target controls are requirements or assessment inputs unless
a future engagement attaches verified evidence. This repository currently
contains no retained production result for this migration path.

## Reference source state

| Area | Reference assumption |
|---|---|
| Organization | Large enterprise with multiple business units, application teams, shared platform/security/network functions, regulated workloads, and 24x7 operations |
| Kubernetes estate | Multiple production and non-production self-managed clusters on premises, VMware, bare metal, or non-EKS infrastructure |
| Application packaging | Workloads already have container images and Kubernetes manifests, Helm releases, Kustomize overlays, operators, or GitOps definitions |
| Scale | Hundreds of deployable workloads and a fleet large enough to require explicit Pod/IP, quota, API-churn, controller, DNS, registry, and support-capacity analysis |
| State | Mix of stateless services, StatefulSets, persistent volumes, databases, queues, caches, batch, and external enterprise dependencies |
| Platform variation | Source-specific CNI, CSI, ingress, DNS, registry, identity, policy, observability, backup, node OS/runtime, and custom controllers |
| Connectivity | Hybrid routing, DNS, firewall/inspection, proxies, private registries, certificate authorities, identity, and data synchronization during coexistence |
| Delivery ownership | Some objects are declared in Git/Helm/pipelines; some live drift and operator-generated state is expected and must be reconciled |
| Migration constraint | Source remains available until target behavior, state, recovery, and operations are proven; big-bang control-plane replacement is not assumed |

The actual source cluster count, Kubernetes versions, namespaces, workloads,
Pods, nodes, objects, traffic, data, and downtime tolerance must be measured.
“Hundreds” describes the reference problem; it is not a migration result.

## Target requirements

### Business

- Reduce self-managed control-plane and node-lifecycle burden without claiming
  that EKS removes workload, data-plane, security, add-on, or application
  responsibilities.
- Preserve or improve approved customer journeys, service objectives, security,
  compliance, recovery, and delivery capability.
- Migrate in bounded waves with visible progress, ownership, rollback, and
  financial controls.
- Avoid copying unsupported source behavior into the managed target.

### Enterprise topology

- Integrate with the approved AWS Organizations, account, identity, network,
  IPAM, DNS, key, certificate, logging, audit, security, and cost foundations.
- Separate production from non-production control planes.
- Use additional account/cluster cells where trust, regulation, Region,
  recovery, lifecycle, quota, scale, or business autonomy requires them.
- Run production cluster cells across multiple Availability Zones and calculate
  Pod, node, IP, storage, load-balancer, control-plane, and quota demand for
  peak, release, maintenance, zone-loss, and recovery scenarios.
- Treat a Kubernetes cluster as the strong isolation boundary; namespaces are
  logical tenancy controls, not equivalent security boundaries.

### Kubernetes compatibility

- Select a target Kubernetes version supported by EKS and test every source API,
  object, client, controller, operator, chart, and admission integration against
  it.
- Remove or convert deprecated APIs and prevent new alpha/beta dependencies from
  becoming unmanaged lifecycle risks.
- Define the target owner and version for every cluster-scoped API and service.
- Rebuild source-independent desired state without status, UIDs, resource
  versions, managed fields, node assignments, allocated addresses, or generated
  credentials.
- Prove behavior after admission and controller reconciliation; API acceptance
  alone is insufficient.

### Platform mappings

| Source concern | Target requirement |
|---|---|
| CNI and Pod addressing | Amazon VPC CNI or an explicitly approved alternative, with per-AZ IP and node-density model |
| Ingress and service exposure | AWS Load Balancer Controller, Gateway implementation, or approved alternative with DNS/TLS/source-IP behavior tested |
| Storage | Supported CSI drivers and StorageClasses with application-consistent data migration, topology, performance, backup, and restore proof |
| Registry | Reachable approved registry/ECR path, immutable digest, replication and pull-capacity design |
| Human identity | Federated AWS access mapped to EKS authorization, least privilege, audit, and break-glass |
| Workload identity | Dedicated ServiceAccounts with EKS Pod Identity or IRSA as appropriate; no dependence on node-wide application credentials |
| Secrets | External authority and target-time population; no secret values in extraction, Git, logs, or migration reports |
| Admission and Pod security | Pod Security Admission and approved policy engine; no `PodSecurityPolicy` migration |
| Cluster autoscaling | EKS-supported node provisioning model selected from workload and lifecycle requirements |
| Observability | Customer-journey, application, dependency, Kubernetes, node, AWS, audit, delivery, and cost signals available before traffic |
| Backup and recovery | Kubernetes configuration plus workload-specific persistent data recovery, with application validation |

### Migration and data

- Define the migration unit from dependency and rollback boundaries, not merely
  namespace count.
- Establish one declared owner for each target object before deployment.
- Use extraction for inventory and comparison; prefer reviewed Git, Helm, or
  pipeline desired state for repeatable deployment.
- Install APIs, CRDs, controllers, policies, storage, networking, identity,
  secrets references, applications, and traffic dependencies in a validated
  order.
- Use application-specific database, object, queue, and volume replication or
  restore mechanisms with quiesce, consistency, integrity, and rollback rules.
- Keep source and target observability active through a representative
  comparison window.
- Move traffic with numeric thresholds and a named decision owner.

### Operations and decommission

- Assign ownership for the EKS control-plane interface, nodes, add-ons, CNI,
  CSI, DNS, ingress/Gateway, policy, identity, observability, backup, upgrades,
  capacity, quotas, vulnerabilities, incidents, and cost.
- Train and test operators on EKS-specific identity, networking, scheduling,
  storage, failure, upgrade, and recovery behavior.
- Retain the source until target traffic, data, recovery, and independent
  operation pass their observation gates.
- Decommission only after all workloads, traffic, data, certificates, secrets,
  integrations, monitoring, backup, licenses, and rollback dependencies are
  cleared and archived according to policy.

## Non-goals

- moving the source control-plane or etcd database into EKS;
- blindly applying every live object to the target;
- preserving source cluster-admin access or unsupported APIs;
- automatically modernizing every application, database, or architecture;
- claiming zero downtime for stateful workloads without a proven data design;
- adopting EKS Auto Mode, GitOps, service mesh, Gateway API, Karpenter, Spot,
  IPv6, or multi-Region architecture without a requirement and test; or
- presenting object count, Pod readiness, or a completed restore job as customer
  outcome evidence.

## Lifecycle traceability

| Requirement | Lifecycle owner |
|---|---|
| Source inventory, ownership, compatibility, data quality, success criteria | [Source cluster assessment](1-source-cluster-assessment-and-success-criteria.md) |
| Target topology, EKS foundation, service mapping, security and architecture decisions | [Target architecture](2-target-architecture-and-compatibility-decisions.md) |
| Reconciliation, ordered deployment, data migration and wave execution | [Migration implementation](3-migration-implementation.md) |
| Dry run, parity, load, failure, traffic, data, rollback and measurement | [Cutover and validation](4-cutover-validation-and-results.md) |
| EKS operating model, handoff, source freeze, archive and decommission | [Operations and decommission](5-operations-and-decommission.md) |

## Entry criteria

Use this solution when the source already runs workloads on Kubernetes and the
primary transformation is from a self-managed cluster platform to Amazon EKS.
If a workload lacks a production container and Kubernetes contract, route it to
the [on-premises monolith modernization solution](../migrate-on-premises-monolith-to-eks/README.md)
or treat containerization as a separately governed prerequisite before this
cluster migration wave.
