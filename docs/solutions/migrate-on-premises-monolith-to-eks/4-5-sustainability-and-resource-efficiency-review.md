# Sustainability and Resource-Efficiency Review

> **Phase:** 4 Validate · **Category:** Modernization
> **Priority topic:** 35
> **Evidence standard:** Review and measurement framework. No carbon-reduction
> or fleet-wide sustainability percentage is claimed.

## Abstract

Sustainability is evaluated with the other AWS Well-Architected pillars. The
goal is to minimize resources required per business outcome without weakening
security, reliability, performance, or recovery.

## Review dimensions

| Dimension | Design questions | Evidence |
|---|---|---|
| Region and placement | Do residency, latency, renewable-energy goals, and dependency locations support the selected Region? | Approved Region decision and data-flow map |
| Demand alignment | Do replicas, nodes, storage, and telemetry scale down when demand falls? | Utilization and business-output time series |
| Software efficiency | Can algorithms, caching, batching, compression, and event design reduce work? | Benchmark per successful transaction or job |
| Compute architecture | Do Graviton, accelerators, or newer instance families improve price-performance and work completed per resource? | Compatible image plus representative benchmark |
| Data lifecycle | Are duplicated, stale, over-replicated, or over-retained data and logs removed safely? | Retention, tiering, deletion, and recovery tests |
| Change process | Are efficiency opportunities reviewed after incidents, releases, and demand changes? | Versioned actions with owner and measured result |

## Measurement contract

```text
workload efficiency = successful business output / resource consumed
resource intensity  = resource consumed / successful business output
```

Resource may be vCPU time, GiB-hours, storage byte-months, network bytes,
accelerator time, or telemetry volume. The comparison preserves workload scope,
time window, demand, correctness, latency, availability, recovery, architecture,
and pricing confounders.

## Improvement sequence

1. Define the business output and reliability/performance guardrails.
2. Establish matched baseline windows and attribute major resource consumers.
3. Remove waste in application requests and retained data before changing node
   purchasing or hardware.
4. Test one change at a time: right-sizing, scaling, consolidation, Graviton,
   storage tiering, sampling, or application optimization.
5. Compare output, resource, latency, error, and recovery results.
6. Keep the change only when the full workload objective improves.

## Guardrails

- scaling to zero is used only where startup and availability objectives allow;
- aggressive consolidation respects disruption budgets and stateful placement;
- shorter retention preserves regulatory, security, and recovery obligations;
- lower telemetry volume preserves incident diagnosis and service objectives;
- cost is not used as a proxy for sustainability without workload-normalized
  resource evidence.

## References

- [AWS Well-Architected Sustainability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/sustainability-pillar/welcome.html)
- [Amazon EKS Cost Optimization](../../practices/aws/eks-bpg/eks-bp-cost-optimization.md)
