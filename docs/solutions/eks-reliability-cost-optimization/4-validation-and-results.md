# Validation and Results

## Abstract

This phase records what the retained evidence supports, workload by workload
and at cluster level, and the gates each change had to pass. It specifies the
cost calculation and the confounders a future percentage claim would have to
control, and documents the recommendation that was rejected.

## What was validated

The retained evidence supports three conclusions:

1. the cluster had a large gap between scheduler-visible requests and observed
   point-in-time usage, especially for CPU;
2. a selected six-replica workload safely completed a rollout after a 90% CPU
   request reduction, while its memory request and limits stayed unchanged; and
3. blindly applying a low VPA CPU target to an HPA-managed service would have
   changed its scaling behavior and was therefore rejected.

It does not support a fleet-wide cost-reduction percentage, a production
Karpenter result, or a claim that every workload was oversized.

## Validation gates

| Gate | Evidence required | Pilot result |
|---|---|---|
| Source review | Owning Helm value and rendered workload | Pass for the selected request |
| Deployment chain | Deploy job, release revision, rollout | Initial pipeline gap found; live CPU rollout separately verified |
| Runtime parity | Effective request equals intended value | Pass for pilot CPU; chart-proposed memory reduction not part of live pilot |
| Readiness | All replicas return Ready | 6 of 6 Ready after ordered rollout |
| Container health | No new OOM/restart/throttle regression | No rollout failure observed; longer service window not retained |
| Service objective | Latency, errors, throughput, and queues within target | Not established for the pilot |
| Scheduler | Pending duration and placement remain acceptable | No terminal pending condition observed |
| Cost | Matched billing/OpenCost periods normalized for demand | Not established |

The distinction matters: rollout readiness proves that Kubernetes converged.
It does not, by itself, prove application performance or monthly savings.

## Selected workload before and after

| Metric | Before | Verified after | Change | Evidence level |
|---|---:|---:|---:|---|
| CPU request per pod | `500m` | `50m` | -90% | Rollout verified |
| Aggregate CPU request, six pods | `3.0` vCPU | `0.3` vCPU | -2.7 vCPU | Rollout verified |
| Memory request per pod | `512Mi` | `512Mi` | No change | Rollout verified |
| CPU/memory limits | `1` vCPU / `1Gi` | `1` vCPU / `1Gi` | No change | Rollout verified |
| Ready replicas | 6 | 6 | No change | Rollout verified |
| p95 latency and error rate | Not retained | Not retained | Unknown | Not outcome verified |
| Allocated workload cost | Not retained | Not retained | Unknown | Not outcome verified |

This pilot was intentionally narrow. Retaining memory and limits made rollback
simple and prevented a CPU result from being confused with a simultaneous
memory change.

## Cluster result table

| Metric | Historical baseline | Verified comparison | Result |
|---|---:|---:|---|
| Running-pod CPU requests | 86.66 vCPU | No matched fleet readback | Opportunity observed; no fleet result |
| Running-pod memory requests | 297.84 GiB | No matched fleet readback | Opportunity observed; no fleet result |
| Point-in-time CPU usage | 4.53 vCPU | Not a valid comparison window | Diagnostic only |
| Point-in-time memory usage | 143.64 GiB | Not a valid comparison window | Diagnostic only |
| Node count | 8-9 across retained snapshots | No post-program matched window | No reduction claimed |
| Monthly infrastructure cost | Not retained with allocation rules | Not available | No savings percentage claimed |
| Cost per tenant/workload | Definition not established | Not available | Roadmap metric |
| Availability and latency | No common baseline retained | Not available | No reliability outcome claimed |

No fleet-wide infrastructure-cost percentage is claimed because the retained
evidence did not contain a complete source, scope, matched billing windows,
exclusions, or tenant-normalized calculation.

## Cost calculation required for a future claim

A publishable result should use matched periods and preserve the calculation:

```text
normalized period cost = eligible infrastructure cost / business-output units
improvement = (baseline unit cost - comparison unit cost) / baseline unit cost
```

Eligible infrastructure cost should state whether it includes:

- EKS cluster and EC2 worker compute;
- EBS volumes and snapshots;
- load balancers and data processing;
- observability and shared platform services;
- Savings Plan, Reserved Instance, or private-rate effects; and
- credits, taxes, support, and one-time migration charges.

Business output might be active-tenant hours, successful requests, completed
jobs, or another stable outcome. AWS Well-Architected guidance similarly frames
cost optimization as improving cost per business outcome, not merely lowering
raw spend. See
[COST02-BP02](https://docs.aws.amazon.com/wellarchitected/latest/framework/cost_govern_usage_goal_target.html)
and
[COST03-BP06](https://docs.aws.amazon.com/wellarchitected/latest/framework/cost_monitor_usage_allocate_outcome.html).

## Confounders to remove

Before attributing a change to right-sizing, the readback must account for:

- tenant, request, and batch-volume growth;
- instance price, purchase-model, or discount changes;
- node-version or instance-family changes;
- added or removed services;
- storage and network growth;
- business-hour or seasonal differences;
- changes in shared-cost allocation; and
- unrelated incident or maintenance windows.

If no stable normalization is possible, report absolute engineering changes
such as released requested vCPU and label financial impact as unknown.

## Load and failure tests for expansion

Every candidate workload should pass:

1. cold start and readiness;
2. steady representative load;
3. burst above the normal peak;
4. HPA scale-out and scale-in;
5. one node drain or equivalent disruption;
6. downstream slowdown or queue growth;
7. memory growth through a full workload cycle; and
8. rollback to the previous source revision.

Node lifecycle changes add tests for provisioning failure, Availability Zone
capacity shortage, Spot interruption, PDB enforcement, topology spread,
persistent-volume attachment, and consolidation.

## Rejected recommendation

The strongest safety finding was `service-b`. A quiet snapshot showed about `6m`
CPU usage and a VPA target of `15m`, but the service used an 80% HPA target.
Reducing its request from `100m` to `15m` would lower the approximate scale
threshold from `80m` to `12m` per pod. The team rejected the direct change until
burst concurrency, latency, readiness delay, and HPA stabilization could be
tested together.

This rejection is a successful optimization outcome: it prevented a usage-only
recommendation from changing the service control loop without evidence.

## Result statement

The defensible result is:

> A measurement-led EKS optimization workflow combined Prometheus, OpenCost,
> recommendation-only Goldilocks/VPA, HPA threshold analysis, Kubernetes/AWS
> inventory, and durable Helm ownership. It identified substantial
> scheduler-visible CPU headroom, completed a six-replica pilot that reduced
> one workload's CPU request by 90% while retaining memory and limits, and
> rejected an unsafe HPA-coupled recommendation. Fleet-wide infrastructure
> savings and Karpenter outcomes remain to be measured.
