# Continuous Optimization

## Operating model

Cost optimization is a recurring platform capability, not a one-time
right-sizing campaign. The steady-state process assigns every recommendation an
owner, an evidence window, a reliability gate, and an expiry date.

| Cadence | Review |
|---|---|
| Daily | SLO breaches, OOM, throttling, pending pods, failed provisioning, and cost anomalies |
| Weekly | New VPA recommendations, HPA saturation/churn, request drift, idle resources, and rollout exceptions |
| Monthly | Cluster/workload cost, unit cost, node utilization, shared-cost allocation, and savings realization |
| Quarterly | Instance families, purchase models, Karpenter policies, storage, resilience tests, and SLO relevance |
| After material change | Rebuild baseline after traffic, runtime, architecture, or autoscaling changes |

## Dashboards

The platform dashboard should connect cost and reliability rather than placing
them in separate views:

| View | Signals |
|---|---|
| Demand | Request rate, throughput, active tenants/jobs, queue depth, seasonality |
| Service | Availability, p50/p95/p99 latency, errors, saturation, retries |
| Containers | Usage and requests, limits, throttling, OOM, restarts, VPA bounds |
| Autoscaling | HPA current/desired/max replicas, threshold, stabilization, events |
| Scheduling | Pending duration, unschedulable reason, topology and volume conflicts |
| Nodes | Allocatable/requested/used resources, pod density, instance family, age, drift |
| Karpenter | Provisioning failures, NodeClaims, disruption decisions, consolidation, NodePool limits |
| Cost | Cluster, namespace, workload, shared platform cost, and cost per business-output unit |

Point-in-time `kubectl top` remains useful for diagnosis, but it cannot replace
the historical dashboard.

## Alerts and budgets

Alerts are designed around action:

- latency or error-budget burn after an optimization;
- new OOM kills, restarts, or sustained CPU throttling;
- HPA at maximum replicas while demand remains above target;
- repeated HPA scale direction changes;
- pods pending beyond the workload's scheduling objective;
- node provisioning or interruption-handler failure;
- voluntary disruption blocked longer than its policy allows;
- NodePool resource limits approaching exhaustion;
- source and effective runtime resources drifting apart;
- workload or namespace spend outside its forecast; and
- unit cost increasing faster than business output.

AWS Budgets and cost-anomaly alerts should route to the accountable cost owner,
while service and capacity alerts route to the platform and application owners.
An alert without an owner and response is only a report.

## Exception workflow

Some workloads legitimately need requests above observed P95. An exception
record contains:

- workload and owner;
- request, limit, HPA, and placement policy covered;
- reason, such as cold-start latency, reserved failover capacity, heap behavior,
  or an unmeasured peak;
- supporting test or incident evidence;
- reliability consequence if removed;
- approval and expiration date; and
- next measurement needed.

Expired exceptions return to review. Policies should detect missing resource
settings, but they should not enforce one global request-to-usage ratio.

## Drift prevention

The durable loop is:

1. generate recommendation-only VPA data;
2. open a review with the observation window and affected HPA threshold;
3. change the owning Helm or IaC source;
4. render and review the diff;
5. deploy through the normal pipeline;
6. verify release revision, rollout, and effective runtime values;
7. observe service and cost gates; and
8. retain the decision and rollback evidence.

Admission policies can require resources to be present. GitOps detects runtime
drift. Neither should automatically apply a low VPA recommendation to a shared
or HPA-managed workload.

## Node lifecycle runbook

### Before enabling a NodePool

- Confirm controller identity, permissions, discovery tags, and interruption
  queue.
- Pin a tested AMI and controller/chart version.
- Verify NodePool resource limits and instance diversity.
- Account for DaemonSet overhead, maximum pod density, topology, and the largest
  pod.
- Verify PDBs for replicated workloads and explicit handling for single-replica
  stateful workloads.
- Confirm persistent-volume zones and `WaitForFirstConsumer` behavior.
- Keep enough managed-node capacity for critical platform components and
  rollback.

### During canary

- Start with fault-tolerant, non-production workloads.
- Permit at most one voluntary node disruption.
- Observe NodeClaims, scheduling latency, consolidation, and service health.
- Run a controlled drain and an interruption exercise.
- Compare a complete workload and cost window before expanding.

### Rollback

1. stop further NodePool sync or voluntary disruption;
2. restore managed-node desired capacity;
3. remove canary selectors/tolerations so pods return to managed nodes;
4. drain and remove canary capacity only after replacements are Ready; and
5. verify workloads, volumes, SLOs, and source/runtime parity.

Amazon EKS guidance recommends broad instance flexibility for Spot and an SQS
interruption queue so Karpenter can react to involuntary events. See
[Karpenter best practices](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html).

## Review template

```text
Workload:
Owner:
Baseline window:
Comparison window:
Demand/business-output change:
Current request/limit/HPA threshold:
VPA lower/target/upper:
Prometheus P50/P95/P99/max:
Proposed source change:
Expected cost effect:
Reliability risks:
Load/failure tests:
Abort conditions:
Rollback revision and owner:
Deployment evidence:
Observed result:
Cost/unit-cost result:
Follow-up date:
```

## Lessons

1. Requests are control inputs, not billing labels.
2. Right-sizing comes before node consolidation because both scheduler and node
   provisioner act on requests.
3. VPA recommendations require workload knowledge, HPA math, and service-level
   evidence.
4. Adding missing requests can be the correct reliability change even when it
   increases apparent reserved capacity.
5. A merge, pipeline check, or Ready rollout proves different things; preserve
   the complete chain.
6. Report realized unit-cost improvement only from matched cost and demand
   periods.
7. A rejected recommendation can be as valuable as an applied saving when it
   prevents an unsafe control-loop change.
