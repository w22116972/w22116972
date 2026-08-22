# Amazon EKS FinOps and Cost Optimization

## Role in the journey

**Category:** Modernization
**Priority topic:** 18
**Evidence status:** A bounded resource-request pilot is retained in the related
solution. No fleet-wide savings percentage or Karpenter savings is claimed.

Cost optimization follows workload correctness and attribution. Lower spend
that increases latency, failed transactions, operator effort, or recovery risk
is not an improvement.

## Optimization order

1. Define business output and service-objective guardrails.
2. Attribute cluster, namespace, workload, storage, network, observability, and
   shared-platform cost.
3. Correct requests and unnecessary application consumption.
4. Remove unused replicas and node capacity with workload and node autoscaling.
5. Select suitable instance families, Graviton where compatible, and storage.
6. Use On-Demand for uncertain or critical baseline, commitments for stable
   measured demand, and Spot for interruption-tolerant variable demand.
7. Review architecture, licenses, data transfer, retention, and managed-service
   alternatives.

## Measurement contract

```text
unit cost = eligible workload cost / successful business output
change    = (baseline unit cost - comparison unit cost) / baseline unit cost
```

The worksheet preserves account, cluster, namespace, workload, dates, demand,
successful output, utilization, requests, replicas, node hours, purchase model,
discounts, credits, support, storage, network, telemetry, tax treatment,
confounders, source queries, and reviewers.

## Change gates

| Change | Validate before expansion | Rollback trigger |
|---|---|---|
| Request reduction | Burst, latency, concurrency, throttling, OOM and HPA behavior | Service-objective or stability regression |
| Graviton | Multi-architecture image, dependency support and benchmark | Correctness or price-performance regression |
| Spot | Interruption handling, instance diversity and checkpoint/idempotency | Recovery exceeds objective or backlog becomes unsafe |
| Karpenter consolidation | PDB, topology, storage and drain behavior | Availability or placement regression |
| Commitment purchase | Stable normalized baseline and ownership horizon | Material demand or architecture uncertainty before purchase |

## Operating cadence

- weekly: anomalies, idle resources, failed jobs, unallocated cost and immediate
  safety issues;
- monthly: unit cost, requests, replicas, storage, data transfer, telemetry, and
  purchase-model coverage;
- quarterly: instance generations, Graviton, Karpenter policies, commitments,
  architecture alternatives, and service-objective relevance.

## References

- [Amazon EKS compute and autoscaling cost guidance](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-compute.html)
- [EKS Reliability and Cost Optimization](../eks-reliability-cost-optimization/README.md)
