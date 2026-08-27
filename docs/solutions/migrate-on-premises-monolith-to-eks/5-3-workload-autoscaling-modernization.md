# Workload Autoscaling Modernization

> **Phase:** 5 Operate · **Category:** Modernization
> **Priority topic:** 17
> **Evidence standard:** HPA, resource, and advisory VPA patterns are evidenced.
> Workload-specific thresholds and scaling results require retained load tests.

## Abstract

Autoscaling is a layered control loop. Application replicas scale from workload
signals; node capacity scales from schedulable demand. Neither layer can create
database connections, external quotas, licenses, or dependency capacity that do
not exist.

## Signal decision

| Mechanism | Use for | Primary risk |
|---|---|---|
| HPA resource metrics | CPU or memory demand correlated with service saturation | Utilization is relative to requests; bad requests create bad scaling |
| HPA custom/external metrics | Concurrency, latency precursor, or another service signal | Metric freshness, cardinality, and unavailable-metric behavior |
| KEDA | Queue, stream, event, schedule, or supported external-system demand | Scale-to-zero startup, authentication, polling, and dependency protection |
| VPA recommendations | Request-right-sizing evidence | Automatic changes can restart Pods and conflict with HPA or availability |
| Scheduled scaling | Predictable peaks or business windows | Forecast drift and persistent excess capacity |

## Control-loop design

```text
business demand
  -> application saturation signal
    -> HPA or KEDA changes replicas
      -> scheduler places Pods
        -> Karpenter or Cluster Autoscaler changes nodes
          -> service and dependency objectives validate the result
```

## Implementation sequence

1. Define steady, burst, startup, cooldown, and dependency limits from a
   representative workload profile.
2. Set requests from observed percentiles, concurrency, and safe headroom.
3. Choose a signal that leads saturation and is available during failure.
4. Configure minimum, maximum, scale-up, scale-down, stabilization, and cooldown
   behavior.
5. Run VPA in recommendation or audit mode before applying request changes.
6. Load-test from minimum capacity, then test metric loss, dependency slowdown,
   queue replay, node shortage, and scale-down disruption.
7. Compare throughput, latency, errors, backlog, replicas, pending time, node
   capacity, and cost per successful output.

## Acceptance and rollback

- scaling begins before the service objective is exhausted;
- maximum replicas respect downstream capacity and account quotas;
- missing or stale metrics produce a known safe behavior;
- scale-down does not violate PDBs, connection drain, jobs, or stateful storage;
- HPA and VPA do not control the same CPU or memory dimension incompatibly;
- rollback restores the prior request and autoscaler revision together.

## References

- [Amazon EKS compute and autoscaling cost guidance](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-compute.html)
- [Amazon EKS Autoscaling Best Practices](../../practices/aws/eks-bpg/eks-bp-autoscaling.md)
