# Amazon EKS Lifecycle and Upgrade Engineering

## Role in the journey

**Category:** Modernization
**Priority topic:** 26
**Evidence status:** Upgrade, add-on, rollout, and rollback practices are retained.
No specific production cluster version or upgrade result is claimed here.

EKS lifecycle management covers the control plane, nodes, managed and
self-managed add-ons, CRDs, admission webhooks, controllers, clients, workloads,
and external integrations. A successful control-plane update is only one gate.

## Lifecycle inventory

| Layer | Compatibility questions | Runtime proof |
|---|---|---|
| Kubernetes APIs | Are removed, deprecated, or changed APIs used in live objects, manifests, Helm output, or controllers? | API scan plus server-side dry run and lower-environment deployment |
| Control plane | Are upgrade insights, quotas, endpoint access, and clients ready? | Healthy API, audit, authentication, and platform tests |
| Nodes | Do kubelet skew, AMI, runtime, CNI, storage, GPU, and architecture remain compatible? | Canary node, scheduling, storage and workload tests |
| Add-ons and CRDs | Are version order, conversion, webhooks, and rollback supported? | Controller conditions, CRD conversion and representative objects |
| Workloads | Do PDBs, probes, topology, resources, policies and dependencies tolerate replacement? | Rollout, disruption, load and failure tests |

## Upgrade sequence

1. Inventory effective versions, ownership, support dates, add-on dependencies,
   APIs, CRDs, clients, node images, and rollback limits.
2. Remove deprecated APIs and incompatible clients before changing the control
   plane.
3. Rehearse in a representative lower environment with the same admission,
   policy, add-on, and workload classes.
4. Upgrade the control plane one supported minor version at a time where
   required, then observe a defined bake period.
5. Upgrade add-ons in their documented order and validate networking, DNS,
   proxy, storage, identity, load balancing, and telemetry.
6. Replace nodes in controlled batches using managed node groups, Karpenter
   drift/expiry, or the selected data-plane mechanism.
7. Validate business journeys, jobs, stateful workloads, autoscaling, recovery,
   and operational tools before closing the change.

## Rollback boundary

Kubernetes and EKS rollback support is version- and mode-specific. Before every
upgrade, record which layers can be reverted, rebuilt, or only moved forward.
Preserve compatible manifests, images, configuration, backups, and N-1 node
capacity where the chosen operating mode permits it.

## Acceptance evidence

- all intended nodes and add-ons converge on approved versions;
- no unsupported APIs or conversion failures remain;
- webhooks cannot deadlock cluster recovery;
- workloads survive drain and reschedule inside objectives;
- rollback or forward-recovery is exercised for each owned layer;
- documentation, IaC, GitOps, inventory, and runtime state agree.

## References

- [Amazon EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
- [Amazon EKS Upgrades](../../practices/aws/eks-bpg/eks-bp-upgrades.md)
- [Amazon EKS Rollback](../../practices/aws/eks-bpg/eks-bp-rollback.md)
