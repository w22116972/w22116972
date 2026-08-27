# Karpenter Compute and Elasticity Modernization

> **Phase:** 5 Operate · **Category:** Modernization
> **Priority topic:** 16
> **Evidence standard:** Karpenter architecture and canary migration are
> designed; fleet-wide deployment, savings, and reliability outcomes are not
> claimed.

## Abstract

Karpenter responds to unschedulable Pods and selects compatible EC2 capacity
from their resource and scheduling requirements. It cannot correct inflated
requests, unsafe affinity, blocked disruption, quota exhaustion, or a saturated
downstream service.

## Provisioning decision

| Model | Prefer when | Operating responsibility |
|---|---|---|
| Managed node groups and Cluster Autoscaler | Capacity shapes are stable and Auto Scaling group boundaries are desirable | Maintain groups, Cluster Autoscaler, AMIs, scaling configuration, and upgrades |
| Self-managed Karpenter | Workloads are diverse or bursty and workload-aware instance selection and consolidation are valuable | Operate Karpenter, IAM, interruption queue, NodePools, EC2NodeClasses, AMIs, limits, and upgrades |
| EKS Auto Mode | AWS-managed compute, networking, load-balancing, and storage operations fit workload and feature requirements | Govern NodePools, workloads, disruption, cost, and compatibility while AWS manages more components |

## Architecture

- retain a small managed node group or another independent capacity boundary for
  Karpenter and critical platform services;
- use broad, approved instance categories, sizes, and Availability Zones;
- separate capacity by interruption tolerance, architecture, accelerator,
  isolation, or compliance only when a real requirement exists;
- pin tested AMIs in production and use drift or expiry through controlled
  batches;
- configure NodePool CPU and memory limits plus budgets and cost alarms;
- enable interruption handling before scheduling fault-tolerant workloads on
  Spot capacity.

## Implementation sequence

1. Baseline pending Pods, requests, node utilization, placement constraints,
   scale latency, interruptions, cost, and service objectives.
2. Correct unsafe requests and validate HPA or event-driven scaling first.
3. Install Karpenter on independent capacity with least-privilege IAM and an
   interruption queue.
4. Create one On-Demand canary NodePool and EC2NodeClass with pinned AMIs,
   approved subnets, security groups, instance families, and resource limits.
5. Move stateless, disruption-tolerant workloads through explicit scheduling
   constraints.
6. Test provisioning, consolidation, drain, PDBs, topology, zonal shortage,
   volume placement, controller loss, and upgrade replacement.
7. Add diversified Spot capacity only after interruption recovery passes.
8. Retire equivalent Cluster Autoscaler groups after rollback dependencies end.

## Acceptance and rollback

- unschedulable Pods receive suitable capacity inside the objective;
- Karpenter does not run solely on nodes whose lifecycle it controls;
- consolidation does not violate availability, storage, or latency objectives;
- NodePool limits and cloud quotas bound runaway capacity;
- nodes use approved AMIs and workload identities;
- rollback reschedules the canary workload to retained managed node capacity
  before deleting its NodePool or EC2NodeClass.

## References

- [Amazon EKS Karpenter best practices](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html)
- [EKS Reliability and Cost Optimization](../eks-reliability-cost-optimization/README.md)
