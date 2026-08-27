# Optimization Implementation

## Abstract

Every optimization moves through the same path:

```text
inventory -> classify -> review -> source change -> lower-environment test
          -> canary -> observation -> expand or rollback -> cost readback
```

The workflow is deliberately slower than changing a live request. A value is
not complete when it is recommended, committed, or merged. It is complete only
when the deployment path ran, the workload rolled out, the effective runtime
value matches source, and the observation window passed.

## Phase 1: inventory and ownership

The inventory joined Kubernetes objects to their top-level workload and owning
Helm values. Each candidate was classified before editing:

| Class | Example signal | Required action |
|---|---|---|
| Oversized | Request consistently far above representative P95 and VPA target | Review smaller request with application owner |
| Missing | Empty resources block | Add conservative request and limit, then observe |
| HPA coupled | Utilization-based HPA reads the resource being changed | Recalculate scale threshold and load test request/HPA together |
| Stateful | Cache, database, queue, local storage, or slow warm-up | Review memory, recovery, quorum, and disruption separately |
| Platform critical | DNS, ingress, autoscaling, GitOps, or storage control plane | Require redundancy and a maintenance window |
| Unowned or abandoned | No confirmed owner or traffic | Establish callers and deletion rollback before removal |
| Insufficient evidence | Short window, missing peak, or conflicting metrics | Collect more evidence; do not optimize yet |

This step also found workloads with no requests at all. Adding explicit values
could increase scheduler-visible demand, but it improved reliability and made
future cost analysis honest.

## Phase 2: recommendation review

For each container, reviewers used this record:

| Field | Review question |
|---|---|
| Demand window | Does the sample include business peaks, jobs, warm-up, and restart behavior? |
| CPU distribution | What are P50, P95, P99, and burst duration? |
| Memory distribution | What are working-set P95, max, and post-restart growth? |
| VPA | Do target and upper bound agree with Prometheus? |
| Runtime | Are JVM heap, caches, thread pools, or native memory relevant? |
| Autoscaling | What absolute usage triggers HPA before and after? |
| Service objective | Which latency, error, throughput, or queue signal can veto the change? |
| Failure mode | What happens on throttling, OOM, restart, node loss, or downstream saturation? |
| Rollback | Which source commit and runtime check restore the previous state? |

Recommendations were rounded to operable values and kept explicit per
environment when demand differed. A low-traffic environment was not allowed to
set a production request by analogy.

## Phase 3: durable source changes

Application settings were changed in their owning Helm values. Shared platform
components were managed by Argo CD, while AWS infrastructure remained in
Terraform. This preserved clear responsibility:

| Layer | Durable owner | Verification |
|---|---|---|
| Container requests, limits, and HPA | Application Helm chart/values | `helm template`, chart tests, effective Deployment/StatefulSet resources |
| Shared platform add-ons | Argo CD Application and values | diff, sync health, rendered resources, rollout |
| Node IAM, interruption queue, network tags | Terraform | plan review, apply evidence, AWS readback |
| Karpenter NodePool and EC2NodeClass | Argo CD after canary approval | manual initial sync, NodeClaim and node readback |

A merge did not count as a deployment. The release chain had to show:

```text
reviewed change -> merge -> deploy job -> Helm revision -> workload rollout
                -> effective resource values -> service observation
```

## Phase 4: selected workload pilot

A six-replica cache workload, `service-a`, provided the first bounded pilot.
Its retained VPA history suggested about `35m` CPU against a `500m` request per
pod. Reviewers selected a conservative `50m` CPU request and kept the `1` CPU
limit to retain burst headroom.

The durable chart change also proposed reducing memory, but the initial live
pilot deliberately retained the existing `512Mi` memory request. This isolated
the CPU variable and avoided combining compressible and non-compressible
resource changes.

| Setting | Before | CPU pilot |
|---|---:|---:|
| CPU request per pod | `500m` | `50m` |
| Aggregate CPU request at six replicas | `3.0` vCPU | `0.3` vCPU |
| CPU limit per pod | `1` vCPU | `1` vCPU |
| Memory request per pod | `512Mi` | `512Mi` |
| Memory limit per pod | `1Gi` | `1Gi` |

The rolling update completed with all six replicas Ready. During the ordered
StatefulSet rollout, temporary reduced readiness and cluster-state warnings
were treated as rollout evidence to interpret, not immediate proof of failure.
The team checked update strategy, events, pod readiness, and final convergence
before closing the change.

The pilot released `2.7` vCPU of scheduler demand, a 90% reduction for this one
request. It did not establish a 90% workload cost saving or a fleet-wide cost
outcome.

## Deployment gap discovered

The pilot exposed an important operational failure mode. The chart value had
been merged, but the first pipeline ran only connectivity checks and did not
create a deployment job. The live Helm revision therefore remained unchanged.
A separately authorized live CPU patch proved the rollout, but it was not
durable by itself.

The corrective rule was:

- record source, pipeline, release, and runtime state separately;
- never describe a merge-only change as deployed;
- persist any successful emergency patch in the owning Helm values; and
- run the normal deployment before declaring drift closed.

## Phase 5: HPA-coupled workload review

For `service-b`, VPA recommended `15m` CPU while the service used a `100m`
request and an 80% HPA target. Direct adoption was rejected because it would
lower the approximate per-pod scale threshold from `80m` to `12m`.

The required test plan was:

1. replay idle, normal, and burst traffic;
2. compare latency, throughput, queue depth, and downstream saturation;
3. inspect HPA desired/current replicas and scaling events;
4. include startup and readiness delay in the burst scenario;
5. verify stabilization windows avoid oscillation; and
6. change the request and HPA policy together if the current target no longer
   represents the intended concurrency.

Until that evidence exists, retaining the higher request is a reliability
decision, not a failure to optimize.

## Phase 6: node lifecycle canary

Right-sizing must precede node optimization because both the scheduler and
Karpenter act on pod requests. The proposed canary sequence is:

1. add disruption protection and topology rules;
2. provision IAM, discovery tags, and an interruption queue through Terraform;
3. install Karpenter with a small managed node group reserved for critical
   platform services;
4. create a capacity-limited canary NodePool with manual initial sync;
5. move only fault-tolerant non-production workloads;
6. test provisioning, drain, consolidation, Spot interruption, and zonal
   volume behavior;
7. compare matched OpenCost windows; and
8. expand one capacity step at a time with at least one full workload cycle
   between steps.

Karpenter, Spot, instance-family changes, and node-disk reduction were not
implemented in the retained evidence and are not included in the result.

## Abort and rollback mechanics

| Trigger | Immediate action | Rollback |
|---|---|---|
| Latency or error objective breach | Stop expansion | Revert the Helm request/HPA pair and redeploy |
| OOM kill or sustained memory pressure | Restore memory headroom | Revert memory request/limit; inspect heap and cache growth |
| Sustained CPU throttling | Restore CPU headroom or remove inappropriate limit | Revert chart values and compare throttle/latency signals |
| HPA churn or saturation | Freeze rollout | Restore prior request and HPA behavior together |
| Pending or unschedulable pods | Restore capacity | Increase managed node capacity or disable canary placement |
| Unsafe consolidation or volume conflict | Stop voluntary disruption | Disable/delete canary NodePool and return workloads to managed nodes |
| Source/runtime mismatch | Do not expand | Run the owning deployment or revert the out-of-band patch |

Each rollout keeps the previous source revision, rendered manifest, effective
runtime values, and a named owner who can execute the rollback.
