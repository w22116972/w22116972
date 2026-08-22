# Amazon EKS Multi-Tenancy and Governance

## Role in the journey

**Category:** Modernization
**Priority topic:** 28
**Evidence status:** Namespace, RBAC, policy, and workload-isolation mechanisms
are evidenced as reusable patterns. No universal hard multi-tenant boundary is
claimed.

The first decision is the required isolation strength. A Kubernetes namespace
is an administrative boundary, not the same security boundary as a separate
cluster or AWS account.

## Isolation decision

| Requirement | Candidate boundary | Typical controls |
|---|---|---|
| Trusted teams sharing a platform | Namespace in a shared cluster | RBAC, default-deny NetworkPolicy, quotas, LimitRanges, Pod Security, workload identity |
| Workloads requiring dedicated capacity | Namespace plus isolated nodes | Taints, tolerations, affinity, dedicated NodePool, storage and security controls |
| Strong tenant or regulatory isolation | Separate cluster and often separate AWS account | Account guardrails, independent keys, networks, identities, logs and operations |
| Large multi-account platform | Centralized or decentralized cluster-account model | AWS Organizations, RAM where approved, role chaining, account/namespace ownership map |

## Governance contract

Every tenant receives an explicit owner, namespace or cluster, workload account,
identity boundary, quota, network policy, data classification, cost allocation,
release mechanism, service objectives, and exit procedure. Platform-wide CRDs,
admission policies, GatewayClasses, storage classes, and cluster roles remain
under platform ownership.

## Implementation sequence

1. Classify tenants by trust, compliance, blast radius, noisy-neighbor risk, and
   data sensitivity.
2. Select namespace, node, cluster, and account boundaries from that model.
3. Start shared namespaces with default-deny network policy and least-privilege
   RBAC; allow only required DNS, platform, and dependency flows.
4. Apply quotas, LimitRanges, Pod Security, workload identities, and approved
   storage and ingress classes.
5. Separate platform and tenant GitOps permissions and prohibit tenant mutation
   of cluster-scoped infrastructure.
6. Test cross-tenant read, write, network, identity, quota, and scheduling
   isolation.
7. Reassess the boundary when trust or regulatory requirements change.

## Acceptance and exit

- unauthorized cross-tenant Kubernetes and AWS API requests fail;
- network tests prove denied and approved flows;
- one tenant cannot consume unbounded shared CPU, memory, storage, or objects;
- cost and service health can be attributed to an owner;
- tenant deletion removes runtime access while preserving required audit and
  recovery records;
- a documented path moves a tenant to a stronger boundary when needed.

## References

- [Amazon EKS tenant isolation best practices](https://docs.aws.amazon.com/eks/latest/best-practices/tenant-isolation.html)
- [Amazon EKS multi-account strategy](https://docs.aws.amazon.com/eks/latest/best-practices/multi-account-strategy.html)
