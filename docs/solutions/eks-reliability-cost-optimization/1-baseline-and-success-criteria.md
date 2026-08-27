# Baseline and Success Criteria

## Abstract

This phase establishes the sanitized historical baseline for a shared Amazon
EKS cluster across a stated evidence window, classifies that evidence before
any claim is made, and states the reliability and cost problem it revealed. It
is a bounded engineering snapshot, not a statement about the cluster's current
state.

## Scope

This case study uses a sanitized historical assessment of `cluster-a`, a shared
Amazon EKS cluster that hosted development, staging, and demonstration
workloads. The retained evidence spans 2026-08-08 through 2026-08-19. It is a
bounded engineering snapshot, not a statement about the cluster's current
state.

## Evidence standard

Evidence is classified before a claim is made:

| Level | Meaning |
|---|---|
| Observed | Captured from Kubernetes, AWS, Prometheus, or billing tooling at a stated time |
| Source verified | Present in reviewed Helm, Argo CD, or Terraform source |
| Rollout verified | The effective live configuration and workload convergence were checked |
| Deployment verified | Source, owning pipeline, release, rollout, and effective runtime state were checked end to end |
| Outcome verified | A before/after window met both cost and service-level criteria |
| Modeled | A planning estimate that has not been realized and is excluded from claimed savings |

The work reached observed and source-verified evidence across the cluster. One
six-replica workload also reached rollout-verified evidence for its CPU-request
change. Its normal deployment chain was not completed in the retained evidence.
Fleet-wide cost savings and Karpenter outcomes remain unverified.

## Problem statement

`cluster-a` consolidated three lifecycle environments and several team-owned
namespaces on one managed node group. The workload mix included stateless APIs,
workers, caches, graph and relational data services, and operator-managed
platform components. Traffic was uneven: many services were quiet most of the
day, while API and AI workloads could burst or hold in-memory state.

The platform had cost visibility through Prometheus and OpenCost, advisory
right-sizing through Goldilocks and Vertical Pod Autoscaler (VPA), and node
scaling through Cluster Autoscaler. Application resource values lived in
separate Helm repositories, so the platform repository could measure and
recommend changes but did not own every deployment.

The consulting objective was therefore not "make every request smaller." It
was to create an auditable path from demand to a safe, durable capacity change.

## Historical baseline

The latest retained inventory contained 317 running pods:

| Signal | CPU | Memory | Interpretation |
|---|---:|---:|---|
| Node allocatable capacity | 143.01 vCPU | 530.22 GiB | Capacity visible to the scheduler after node reservations |
| Running-pod requests | 86.66 vCPU | 297.84 GiB | 61% CPU and 56% memory of allocatable capacity |
| Point-in-time usage | 4.53 vCPU | 143.64 GiB | 3% CPU and 27% memory of allocatable capacity |
| Request-to-usage ratio | 19.1x | 2.1x | Triage signal only; not a right-sizing value |

The cluster moved between eight and nine nodes across the retained snapshots.
The later node-group snapshot was healthy at nine desired nodes, with a minimum
of five and maximum of eleven. All nodes used one general-purpose instance
family and On-Demand capacity.

These numbers revealed opportunity but did not prove waste by themselves:

- point-in-time CPU can miss peaks, initialization, and batch windows;
- memory is non-compressible and may include caches that grow with traffic;
- running-pod aggregation excludes completed jobs and does not reproduce every
  scheduler decision;
- requests are also reliability and isolation controls; and
- node count changes only after pod requests, topology, daemon overhead, pod
  density, and volume constraints permit safe packing.

## Measurement model

| Signal | Source | Window | Aggregation | Limitation | Decision supported |
|---|---|---|---|---|---|
| CPU usage | Prometheus counter rate | 14-30 days plus peak replay | P50, P95, P99 by container and workload | Sampling can miss very short bursts | CPU request candidate and burst headroom |
| Memory working set | Prometheus | At least one workload cycle | P50, P95, max by container | Cache growth and restart peaks need separate review | Memory request and limit candidate |
| VPA bounds | Goldilocks/VPA | Recommender history | Lower, target, and upper by container | Usage-derived; does not know latency or concurrency intent | Independent recommendation, never automatic approval |
| Pod requests | Kubernetes API | Inventory and post-rollout snapshots | Running pods by owner and namespace | Completed jobs and pending replacements require separate handling | Scheduler-visible demand |
| HPA behavior | Kubernetes API and metrics | Peak and rollout windows | Current/desired replicas, target utilization, events | Utilization targets depend on requests | Scaling threshold and stability decision |
| Node capacity | Kubernetes and AWS APIs | Same baseline/readback dates | Allocatable, instance type, node count, age | Does not include all non-node charges | Packing and lifecycle decision |
| Service health | Metrics and load tests | Baseline, canary, and observation window | Availability, p95/p99 latency, errors, throughput, queue depth | Requires workload-specific SLOs | Abort or expand decision |
| Cost | OpenCost and AWS billing data | Matched billing periods | Cluster, namespace, workload, and shared-cost allocation | Pricing changes and allocation rules can confound comparison | Realized savings and unit cost |
| Business output | Tenant or workload telemetry | Same cost period | Cost per active tenant, request, or completed job | Needs stable definitions and ownership | Efficiency rather than raw spend |

## Ownership and constraints

| Area | Accountable owner | Constraint |
|---|---|---|
| Application requests, limits, and HPA | Service owner | Must preserve latency, throughput, and startup behavior |
| Shared add-ons and advisory VPA | Platform team | Recommendations remain `updateMode: Off` |
| Node group and future NodePools | Platform and cloud owners | Stateful, topology, and disruption gates apply |
| Billing allocation | FinOps or cloud owner | Shared costs and discounts need a stable allocation rule |
| Release approval | Service owner plus platform reviewer | Source merge alone is not deployment evidence |

The cluster contained many single-replica stateful workloads, a small number of
PodDisruptionBudgets, zonal persistent volumes, and a single instance family.
Those facts made aggressive consolidation unsafe until disruption protection
and placement constraints were addressed.

## Success criteria

The program uses gates rather than a single cost target:

| Dimension | Acceptance criterion |
|---|---|
| Cost | Matched-period infrastructure cost and unit cost decline after removing pricing and demand confounders |
| Requests | Scheduler-visible CPU/memory fall only for reviewed candidates; missing requests are corrected even if totals rise |
| Availability | No SLO regression during canary and observation window |
| Latency and throughput | p95/p99 latency, throughput, and queue depth remain within the service's accepted envelope |
| Container health | No new OOM kills, sustained throttling, crash loops, or abnormal restart increase |
| Scheduling | No material increase in pending-pod or unschedulable duration |
| Autoscaling | HPA responds to load without oscillation, saturation, or suppressed scale-out |
| Node lifecycle | A controlled drain or interruption respects PDB, topology, and storage constraints |
| Durability | Helm/IaC source, deployment job, rollout, and effective runtime values agree |
| Rollback | A reviewed revert restores the previous request or capacity policy within the agreed recovery window |

No fleet-wide percentage is reported unless all of these gates and the cost
calculation pass for the same comparison period.
