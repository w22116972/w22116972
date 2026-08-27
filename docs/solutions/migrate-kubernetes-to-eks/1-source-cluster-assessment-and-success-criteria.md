# Source Cluster Assessment and Success Criteria

## Abstract

This phase reconciles three views of the source, because a Kubernetes API
inventory alone cannot explain what should exist, which controller owns it, or
how it recovers: declared desired state, effective runtime state, and owner
knowledge. It covers fleet inventory, ownership and drift, workload grouping,
version and API compatibility, CRDs and operators, networking, storage,
identity, and capacity, then states what a complete assessment must produce.

## Assessment principle

A Kubernetes API inventory is necessary but incomplete. Assessment must explain
what should exist, what actually exists, which controller owns it, which
external systems make it work, how it changes, and how it can be recovered.

Use three evidence views:

1. **Declared desired state** — Git, Helm, Kustomize, Terraform, pipelines,
   operator definitions, registry and secret authorities.
2. **Effective runtime state** — live objects, controller status, nodes, events,
   traffic, data, policy, telemetry, and observed dependencies.
3. **Owner knowledge** — business behavior, exceptional procedures, maintenance,
   recovery, vendor and operational dependencies not visible through APIs.

Differences become explicit drift records. Live state is not automatically
correct, and repository state is not automatically deployed.

## Evidence and sensitivity boundary

Discovery is read-only. Private collection may include names and configuration
needed for migration, but public documentation uses symbolic resources. Secret
values, tokens, private keys, kubeconfigs, credentials, personal data, internal
addresses, raw logs, and full production manifests are never published.

Extraction artifacts are protected because ConfigMaps, annotations, environment
variables, image references, RBAC, routes, and dependency names can disclose
sensitive architecture even when Secret values are excluded.

## Source fleet inventory

| Domain | Capture | Why it matters |
|---|---|---|
| Cluster | Distribution, version, API endpoint model, control-plane ownership, etcd/backup model, feature gates, admission config | Establish target version gap and unsupported behavior |
| Nodes | OS, architecture, runtime, kubelet version/config, labels, taints, capacity, allocatable, devices, kernel, cgroups, local disks | Detect scheduling, runtime and node-feature dependencies |
| Workloads | Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, replicas, strategies, probes, resources, security context | Size and classify migration behavior |
| APIs | API groups/versions, deprecated requests, CRDs, conversion/defaulting webhooks, custom resources | Determine target API and controller prerequisites |
| Ownership | Helm release, Git path, Kustomize overlay, operator, Terraform, pipeline or manual owner | Prevent multiple writers and raw-object replay |
| Networking | Services, Ingresses, Gateway APIs, NetworkPolicies, CNI, Pod/Service CIDRs, DNS, source IP, egress, proxies, load balancers | Map connectivity and address behavior |
| Storage | CSI/in-tree provisioners, StorageClasses, PV/PVC, access mode, topology, reclaim, snapshots, performance, local storage | Separate object recreation from data migration |
| Identity | Human authentication, RBAC, ServiceAccounts, token behavior, external identity, cloud credentials | Rebuild least privilege for AWS and EKS |
| Policy/security | Admission webhooks, policy engines, Pod security, image policy, privileged/host access, runtime agents | Identify blockers and target controls |
| Configuration | ConfigMaps, secret metadata/authority, certificates, environment values, reload/rotation | Plan secure target population and lifecycle |
| Cluster services | DNS, ingress/Gateway, autoscaling, metrics, logs, traces, policy, backup, registry helpers, certificate and secret operators | Rebuild dependencies before applications |
| External dependencies | Databases, queues, object stores, APIs, identity, DNS, certificates, file shares, licenses, partner allowlists | Prove hybrid and target request paths |
| Delivery | Source, build, image, chart, GitOps/pipeline revision, promotion, rollback | Reconcile declared and running identity |
| Operations | Owners, SLOs, incidents, runbooks, backups, maintenance, upgrades, support, cost | Design handoff and acceptance |

## Inventory reconciliation

For every significant object or logical application, record:

| Field | Meaning |
|---|---|
| Logical owner | Team accountable for behavior and retirement |
| Write owner | Helm, GitOps, operator, pipeline, Terraform, platform controller, or approved manual process |
| Declared location | Repository/chart/module and revision where desired state lives |
| Effective identity | Live kind, namespace, name, API version, image digest, configuration identity |
| Drift class | Expected defaulting, generated state, emergency change, stale declaration, or unexplained drift |
| Target disposition | Rebuild, transform, replace, retain external, retire, or investigate |
| Evidence | Source export, rendered object, live readback, controller status, test, and owner review |

Do not deploy extracted controller-owned children as independent desired state.
Examples include ReplicaSets owned by Deployments, Pods owned by controllers,
Endpoints/EndpointSlices owned by Services/controllers, generated certificate
Secrets, service-account tokens, and operator-generated custom resources.

## Workload grouping and criticality

The migration unit is dependency-complete and reversible. It can be an
application spanning namespaces, a tenant, a service group, or a namespace when
namespace boundaries actually align with ownership and dependencies.

| Attribute | Assessment |
|---|---|
| Business | Capability, product owner, criticality, customer impact, critical periods |
| Service objectives | SLI/SLO, demand, latency, error, saturation, RTO, RPO, downtime allowance |
| Runtime | Stateless/stateful, replicas, autoscaling, singleton behavior, Jobs, startup and drain |
| Dependencies | Upstream, downstream, synchronous, asynchronous, data, identity, DNS, operator, platform |
| State | System of record, writers/readers, consistency, replication, backup, restore, reconciliation |
| Portability | API, image, node, architecture, device, privilege, network, storage and platform assumptions |
| Release | Source owner, artifact, configuration, schema order, rollback, maintenance window |
| Operations | On-call, dashboard, alerts, runbooks, support, failure history, capacity and cost owner |

## Kubernetes version and API compatibility

1. Record source control-plane, node, client, controller and chart compatibility.
2. Select the target EKS Kubernetes version from current support and enterprise
   lifecycle policy; do not embed a soon-to-expire version in the case study.
3. Locate deprecated and removed API use through manifests, rendered Helm,
   GitOps state, audit/metrics, controller clients, and live objects.
4. Convert declarations to supported APIs and review semantic changes, defaults,
   selectors, policies and fields—not only `apiVersion` text.
5. Test manifests and clients against the target API, admission chain and CRDs.
6. Record every alpha/beta API or feature dependency with owner, target support,
   upgrade impact, fallback and removal plan.

`PodSecurityPolicy` is a legacy API and is not a migration prerequisite. Source
policy intent is translated to Pod Security Admission and/or a supported policy
engine. Workloads that fail the target policy receive remediation or a bounded,
approved exception; the old policy object is not copied.

## CRD, operator, and webhook assessment

For each CRD/operator pair, capture:

- source, owner, vendor/project support and release version;
- target Kubernetes/EKS compatibility and required AWS permissions;
- CRD versions, storage version, conversion/defaulting, schema and finalizers;
- cluster-scoped resources, leader election, replicas and disruption behavior;
- webhook certificates, Services, endpoint readiness, timeout and failure policy;
- custom resource count, order, external effects and deletion semantics;
- backup/restore and reconcile behavior; and
- upgrade, rollback and removal procedure.

CRDs are installed before controllers only when the vendor procedure supports
that order; controllers must be healthy before dependent custom resources are
expected to reconcile. A custom resource existing in the API is not proof that
its external cloud, storage, network or application effects exist.

## Networking and DNS assessment

| Concern | Questions |
|---|---|
| Address space | Do source Pod, Service, node, enterprise and target VPC CIDRs overlap? Which flows require routability during coexistence? |
| Pod networking | Which CNI features, NetworkPolicy semantics, MTU, encapsulation, source IP and egress behavior are assumed? |
| Service discovery | Which cluster domains, search paths, stub domains, NodeLocal DNS behavior, headless Services and direct Pod records are used? |
| North-south | Which Ingress/Gateway APIs, controllers, annotations, static IPs, TLS, client IP, health checks, timeouts and protocols are required? |
| East-west | Which cross-namespace, cross-cluster, mesh, mTLS, service export or direct Pod dependencies exist? |
| Egress | Which proxies, NAT, inspection, allowlists, certificates and fixed-source-IP requirements apply? |
| Capacity | What are connections, packets, bandwidth, DNS QPS, load-balancer targets, CNI warm addresses and per-AZ subnet demand? |

Connectivity tests cover both directions during coexistence: source-to-target,
target-to-source, target-to-enterprise dependencies, and external clients to
each cluster. NetworkPolicy behavior is tested with allowed and denied flows.

## Storage and data assessment

Kubernetes storage objects describe claims and attachment; they do not move
application-consistent data.

For every stateful workload record:

- application and data owner;
- data class, volume, growth and change rate;
- StorageClass, provisioner, access/volume mode, topology and reclaim policy;
- filesystem/database consistency requirements and writer behavior;
- IOPS, throughput, latency, attachment, mount, UID/GID and encryption needs;
- snapshot, backup, replication, export/import or application migration method;
- quiesce and synchronization window;
- checksum, row/object count, schema, ordering and application invariants;
- RTO/RPO and recovery dependency ordering; and
- last reversible point, rollback direction and split-brain protection.

Source in-tree provisioners are mapped to supported CSI drivers, but that mapping
only recreates future provisioning behavior. Existing PV data needs a separate
tested migration mechanism.

## Security and identity assessment

- map human identities and cluster-admin paths to federated AWS access and EKS
  authorization;
- review RBAC for privilege escalation, wildcards, impersonation, token creation,
  secret access and cluster-scoped grants rather than copying it verbatim;
- inventory every workload cloud/API permission and map it to EKS Pod Identity
  or IRSA where appropriate;
- identify node metadata or static-credential dependencies and remove them;
- map secret metadata to the target secret authority, rotation, reload,
  revocation and audit process without extracting values;
- classify privileged, host namespace, hostPath, device, capability, root,
  writable-filesystem and seccomp requirements;
- map image provenance, signing, SBOM, registry and runtime security controls;
  and
- define control-plane, AWS API, workload, network and security-event evidence.

## Capacity, scale, and quota assessment

Use the shared [enterprise EKS readiness assessment](../migrate-on-premises-monolith-to-eks/1-1-enterprise-assessment.md)
for the full Pod/node/IP and quota model. This migration additionally records
source workload behavior so target capacity is not derived from requested
resources alone:

- steady, peak, batch, release, maintenance, failure and recovery Pods;
- source CPU, memory, ephemeral storage, network, connection and volume use;
- source overcommit, missing requests, node-local dependencies and noisy-neighbor
  effects;
- object count and create/update/delete churn;
- controller, webhook, monitoring and GitOps API load;
- image size, registry bandwidth and concurrent pulls;
- target instance, architecture, `maxPods`, CNI/IP, per-AZ subnet and storage
  attachment limits; and
- AWS regional quotas, API throttles, downstream limits and approved fallback.

## Success criteria

| Objective | Required evidence |
|---|---|
| Complete scope | Reconciled declared/live/owner inventory with expected, migrate, replace, retain, retire and investigate dispositions |
| Compatible target | No unresolved removed API, unsupported controller, unknown cluster-scoped owner, or untested node/platform dependency |
| Functional parity | Contract, integration and critical user journeys pass on source baseline and target candidate |
| Performance and scale | Target meets service objectives under matched demand, rollout, node/zone failure and recovery scenarios |
| Security | Required actions succeed, unauthorized actions fail, policy exceptions are approved and expiring, secret values remain protected |
| Data integrity | System-specific consistency, reconciliation, read/write, backup and restore tests pass inside RTO/RPO |
| Cutover safety | Numeric abort gates, observation period, traffic owner, source capacity and data-aware rollback are tested |
| Operability | Customer team independently deploys, diagnoses, scales, recovers and rolls back target workloads and platform components |
| Financial control | Source, target, dual-run and decommission cost use documented allocation and matched demand |

## Primary risks

| Risk | Mitigation and gate |
|---|---|
| Live extraction becomes unsafe desired state | Reconcile to owners and declared sources; filter generated/status/security-sensitive fields; render and dry-run |
| Hidden cluster service is missing | Map APIs and runtime effects to controllers; prove target reconciliation before dependent workloads |
| Source and target APIs differ semantically | Test supported declarations and controllers against target API/admission, not text conversion alone |
| Storage objects recreate without data | Separate CSI mapping from application-consistent data migration and validation |
| Both clusters write incompatible state | Define writer authority, synchronization, fencing and last reversible point before cutover |
| RBAC or credentials are over-copied | Rebuild least privilege and validate allowed/denied actions |
| DNS or ingress cutover hides client behavior | Test TTL, caching, TLS, source IP, health, long-lived connections and rollback |
| Target capacity is based on misleading requests | Compare observed demand, load tests, failure scenarios, CNI/IP, storage and quotas |
| Source is retired too early | Require traffic, dependency, data, recovery, observation and operations exit evidence |

## Exit criteria
Before target application deployment, retain approved versions of:

1. source fleet, object, ownership and drift inventory;
2. business/application grouping and dependency map;
3. target version, API, CRD, operator, webhook and node compatibility matrix;
4. CNI, CIDR, DNS, ingress/Gateway, egress and coexistence network map;
5. CSI, PV, data synchronization, integrity, RTO/RPO and rollback matrix;
6. human/workload identity, RBAC, secrets and policy mapping;
7. source/target capacity, CNI/IP, quota and failure-state model;
8. platform-service build order and ownership matrix;
9. migration wave score, entry and exit criteria, traffic unit and rollback packet;
10. baseline functional, performance, reliability, security, recovery and cost
    measurements; and
11. risks, assumptions, exceptions, actions, owners and decision deadlines.

Assessment exits only when every first-wave dependency has an owner and target
disposition; source and target state are reconcilable; compatibility and data
methods are testable; target prerequisites and quotas are available; and every
unknown that can affect production has a decision date and safe fallback.

## References

- [AWS self-managed Kubernetes to EKS pre-migration checklist](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/pre-migration-checklist.html)
- [Kubernetes deprecated API migration guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [Amazon EKS known limits and service quotas](https://docs.aws.amazon.com/eks/latest/best-practices/known_limits_and_service_quotas.html)
