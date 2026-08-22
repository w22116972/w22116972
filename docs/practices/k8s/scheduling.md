# Kubernetes Scheduling、Preemption 與 Eviction 面試指南

Kubernetes scheduling 的核心不是「找一台 CPU 還有空間的 node」，而是把 Pod 的 declared intent、cluster policy、resource availability 與 disruption cost 轉成 placement 決策。`kube-scheduler` 只為尚未綁定的 Pod 選擇 node；真正的 resource enforcement 與 node-pressure eviction 由 kubelet 執行，而 voluntary disruption 通常透過 Eviction API 協調 PodDisruptionBudget（PDB）。

## 先掌握完整 lifecycle

```text
Pod admitted
  -> schedulingGates cleared when prerequisites are ready
  -> active scheduling queue, ordered by priority and QueueSort
  -> Filter: remove infeasible nodes
  -> PostFilter: optionally evaluate preemption
  -> Score: rank feasible nodes
  -> Reserve / Permit / PreBind
  -> Bind: write spec.nodeName through the API
  -> kubelet starts and enforces the Pod
  -> later disruption may come from preemption, node pressure, taints, or Eviction API
```

| 問題 | 決策者 | 關鍵事實 |
| --- | --- | --- |
| 新 Pod 應放在哪個 node | `kube-scheduler` | 主要依 requests、constraints 與 cached cluster state，不依即時使用率 |
| Container 是否超過 runtime limit | kubelet、container runtime、kernel | scheduler 不執行 runtime isolation |
| 高 priority Pod 是否移除低 priority Pod | scheduler preemption | PDB 僅 best effort；preemption 不會創造 capacity |
| Node resource pressure 要移除誰 | kubelet | 不遵守 PDB；hard threshold 使用 `0s` grace period |
| Planned drain 是否可移除 Pod | API server 的 Eviction subresource | 遵守 PDB，可能回傳 `429` |
| `NoExecute` taint 是否移除 Pod | node lifecycle/taint manager 與 kubelet | 取決於 toleration 與 `tolerationSeconds` |

## Scheduler pipeline 與 Scheduling Framework

最重要的基本流程是 **Filter → Score → Bind**：

- Filter 判斷 node 是否 feasible，例如 resources、node affinity、taints、volume topology 與 ports。
- 若沒有 feasible node，Pod 保持 Pending；PostFilter plugin 可以嘗試 preemption，但不是每個 unschedulable 原因都能靠移除其他 Pods 解決。
- Score 只比較通過 Filter 的 nodes；最高分 node 被選中，同分時由 scheduler 選擇其中之一。
- Scheduling cycle 為 serial；binding cycle 可並行。`Reserve`、`Permit`、`PreBind` 等 extension points 讓 plugins 參與決策或在失敗時 rollback。

Scheduling Framework 的 profiles 與 plugins 是目前的擴充方式。不要再採用舊式 Scheduling Policy、Predicates/Priorities 或 `kubescheduler.config.k8s.io/v1alpha1` 範例；目前 configuration API 使用 `kubescheduler.config.k8s.io/v1`。

## Requests、Pod Overhead 與 Scheduling Readiness

Scheduler 以 requests 而不是即時 metrics 判斷容量。CPU/memory request 過高會造成不必要的 Pending 與擴容，過低則讓 node 被 overcommit，增加 throttling、OOM 與 eviction 風險。

使用 sandboxed runtime 時，`RuntimeClass.overhead.podFixed` 會在 admission 時加入 `.spec.overhead`。這個 overhead 會計入 ResourceQuota、scheduling、Pod cgroup sizing 與 eviction ranking；應依實測 runtime overhead 設定，不要直接複製範例數值。

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: sandboxed
handler: sandboxed
overhead:
  podFixed:
    cpu: 250m
    memory: 120Mi
```

若 Pod 在 device claim、network attachment 或外部 controller 準備完成前根本不該被排程，可在建立時加入 `.spec.schedulingGates`。Gate 只能在 create 時加入，之後只能移除，不能新增；清除後 Pod 才進入 active scheduling queue。這可避免 scheduler 與 node autoscaler 對尚未 ready 的 Pod 反覆做無效計算，但 gate owner 必須具備 timeout、reconciliation 與 observability，避免永久 `SchedulingGated`。

## Node selection：由簡到複雜

選擇機制應使用最小必要約束：

1. `nodeSelector`：所有 labels 都必須符合，適合簡單 hard requirement。
2. Node affinity：`requiredDuringSchedulingIgnoredDuringExecution` 是 Filter 條件；`preferredDuringSchedulingIgnoredDuringExecution` 只影響 Score。
3. Inter-Pod affinity/anti-affinity：依其他 Pods 的 labels 與 topology placement，表達 co-location 或 separation。
4. `.spec.nodeName`：直接繞過 scheduler，只適合少數受控用途；node 不存在或 kubelet 拒絕時 Pod 仍會失敗。

`IgnoredDuringExecution` 表示 Pod 綁定後 node label 改變不會自動驅逐 Pod。多個 selector/affinity 條件通常會形成 AND，過多 hard constraints 很容易讓 Pod 永久 Pending。大型 cluster 的 inter-Pod affinity/anti-affinity 計算成本也較高，且所有 nodes 必須有一致的 `topologyKey` labels。

不要讓不受信任的 kubelet 能自行加上隔離用 label。需要 security/isolation guarantee 時，使用 NodeRestriction admission plugin 保護的 `node-restriction.kubernetes.io/` label namespace，並由受信任的管理流程套用 labels。

## Topology spread 與 availability

Pod topology spread constraints 用 `maxSkew` 控制符合 `labelSelector` 的 Pods 在 eligible topology domains 間的差距。`topologyKey` 常用 `topology.kubernetes.io/zone` 或 `kubernetes.io/hostname`。

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    minDomains: 3
    labelSelector:
      matchLabels:
        app: api
```

- `DoNotSchedule` 是 hard constraint；`ScheduleAnyway` 是 soft preference。
- `minDomains` 只可搭配 `DoNotSchedule`。eligible domains 少於它時，global minimum 以 0 計算，可能使 Pod 保持 Pending。
- `labelSelector` 應與 workload template labels 相符，否則新 Pod 可能沒有被計入 skew。
- Spread 只能改善 placement，不等於 HA；仍需足夠 replicas、multi-zone capacity、PDB、readiness 與 storage/network failure-domain 設計。
- Pod anti-affinity 適合「不要共置」規則；topology spread 更適合控制多個 domains 的容許偏差。

## Taints 與 tolerations

Taint 排斥 Pod，toleration 只代表「允許」，不會吸引 Pod 到該 node。Dedicated node pool 通常同時使用 taint/toleration 與 node affinity，避免專用 workload 落到一般 nodes。

| Effect | 新 Pod | 已執行 Pod |
| --- | --- | --- |
| `NoSchedule` | 不排入，除非容忍 | 不主動驅逐 |
| `PreferNoSchedule` | 盡量避免 | 不主動驅逐 |
| `NoExecute` | 不排入，除非容忍 | 未容忍者被驅逐；可用 `tolerationSeconds` 延後 |

Kubernetes 會為 node conditions 對應 taints，並為一般 Pods 自動加入有限時間的 `not-ready` 與 `unreachable` tolerations。不要為所有 workloads 加上無限制的 broad toleration，否則會破壞 isolation 與 failure recovery。

## DRA 與 resource bin packing

Dynamic Resource Allocation（DRA）的核心 device allocation 在 v1.35 已 stable。Pod 可透過 ResourceClaim/ResourceClaimTemplate 表達 device requirements，scheduler 把 claim allocation、node placement 與 device availability 一起評估。不要把 DRA 等同於 CPU/memory requests；DeviceClass、ResourceSlice、driver、RBAC 與 admin access 各有不同 trust boundary。非 GA extensions 不屬於本 baseline，security-related details 見 [Kubernetes Security Preview Features](security-preview-features.md)。

`NodeResourcesFit` 的 `MostAllocated` 或 `RequestedToCapacityRatio` 可依 **requested-to-capacity ratio** 做 bin packing，讓低利用 nodes 更容易 consolidation。這不是依 runtime usage 即時塞滿 node；若 requests 不準確，placement 一樣會失真。

Production tuning 必須同時考慮：

- workload burst 與 system/kube reserved headroom；
- zone、volume、GPU/device fragmentation；
- autoscaler consolidation 與 replacement latency；
- failure blast radius，以及單一 node 故障時是否仍符合 SLO。

## Priority 與 preemption

`PriorityClass` 是 cluster-scoped。較高 priority 會讓 Pending Pod 在 queue 中較早被考慮；當沒有 feasible node 時，default preemption 可能移除較低 priority Pods，讓 preemptor 能在某一個 node 上排入。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: important-nonpreempting
value: 100000
globalDefault: false
preemptionPolicy: Never
description: Queue ahead of lower-priority work without evicting it.
```

- `preemptionPolicy: Never` 保留 queue priority，但不主動 preempt；它本身仍可能被更高 priority Pod preempt。
- Preemption 只移除必要的 lower-priority victims，且 victim termination 期間不保證 capacity 仍保留給原 preemptor。
- PDB 在 scheduler preemption 中只是 best effort；找不到不違反 PDB 的 victims 時仍可能 preempt。
- 若 blocker 是 node affinity、taint、volume topology、port conflict 或缺少 device，移除低 priority Pods 未必有幫助。
- Priority 應配合 admission 與 RBAC 管理；過多高 priority workloads 會讓 priority 失去意義，甚至造成 cascading disruption。

## 三種 eviction 不可混淆

| 機制 | 觸發者 | PDB | Grace period | 常見用途 |
| --- | --- | --- | --- | --- |
| Scheduler preemption | `kube-scheduler` | Best effort | 使用 victim termination grace | 為高 priority Pending Pod 騰出空間 |
| Node-pressure eviction | kubelet | 不遵守 | Hard threshold 為 `0s`；soft threshold 受 kubelet max grace 限制 | 保護 node 避免 memory/disk/PID starvation |
| API-initiated eviction | drain、autoscaler 或 API client | 遵守 | 正常 Pod termination flow | Planned maintenance 與 voluntary disruption |

Eviction API 建立 `policy/v1` Eviction subresource；允許時回 `200`，PDB 暫時不允許通常回 `429`。直接 `DELETE Pod` 會繞過 PDB，因此自動化 maintenance 應優先使用 Eviction API、處理 retry/backoff，並在強制刪除前確認應用風險。

Node-pressure eviction 先嘗試 node-level reclaim，例如清除 dead containers 或 unused images，再依下列原則挑 victims：是否超過 requests、Pod Priority、使用量相對 requests。它不是單純按 QoS class 排序，而且 disk/PID 沒有相同的 request-based 模型。監控 `memory.available`、`nodefs`/`imagefs`/`containerfs` availability 與 inodes、`pid.available`，並為 soft thresholds 設定 grace、為 hard thresholds 保留足夠安全邊界。

不要使用已 deprecated 的 kubelet dead-container garbage-collection flags；使用 eviction thresholds 與 runtime/image garbage collection 的現行機制。

## Scheduler performance tuning

`percentageOfNodesToScore` 讓 scheduler 在找到足夠 feasible nodes 後停止搜尋，在大型 cluster 以 placement accuracy 換 scheduling latency。預設值會依 cluster size 計算；數百 nodes 以下通常應保留預設，低於 10% 容易錯過較佳 placement。

無論何種 tuning，都應先以 scheduler metrics/profile 證明 bottleneck，再比較 scheduling latency、attempts、unschedulable Pods、plugin duration 與 placement quality。Managed Kubernetes 通常不能直接修改 control-plane scheduler configuration，應先確認 provider 支援邊界。

## Preview scheduling features

PodGroup、Gang Scheduling、Topology-Aware Scheduling、Workload-Aware Preemption 與其他尚未成熟的 scheduler capabilities 不屬於本 baseline。Alpha group scheduling 見 [Kubernetes Alpha Scheduling Features](scheduling-alpha-features.md)；beta scheduler optimization 見 [Kubernetes Beta Scheduling Features](scheduling-beta-features.md)。

## Pending 與 eviction 的排查順序

1. `kubectl describe pod` 檢查 `PodScheduled` condition、events、`SchedulingGated` 與 nominated node。
2. 驗證 requests/overhead 是否能 fit，並比較 node allocatable，而非只看即時 `kubectl top`。
3. 逐項檢查 nodeSelector/affinity、taints/tolerations、topology spread、ports、PVC/volume topology 與 DRA claims。
4. 檢查 scheduler queue、latency、plugin metrics/logs，以及是否有 webhook/controller 持有 gate。
5. 若發生 preemption，核對 PriorityClass、victims、PDB 與 termination time；不要假設 nominated node 就一定成功 bind。
6. 若發生 eviction，先辨識來源是 kubelet pressure、Eviction API、taint 還是 scheduler preemption，再查對應 events、node conditions 與 audit records。
7. 最後才調整 hard constraints、priority 或 eviction thresholds；先以 dry run、staging load 與 rollback plan 驗證。

## 常見面試題

### Toleration 與 node affinity 有何差異？

Toleration 允許 Pod 進入帶 taint 的 node，但不會把它吸引過去；node affinity 則選擇符合 labels 的 nodes。Dedicated pool 通常兩者並用。

### Topology spread 與 Pod anti-affinity 如何選？

Anti-affinity 表達 Pods 之間不應共置，hard anti-affinity 常限制每個 domain 最多一個；topology spread 直接控制跨多個 domains 的 skew，對多 replicas 的 availability 通常更具表達力。

### PDB 是否能阻止所有 eviction？

不能。Eviction API 會遵守 PDB，scheduler preemption 只有 best effort，node-pressure eviction 不遵守 PDB；直接刪除 Pod 也不受 PDB 保護。

### 為何 requests 是 scheduling reliability 的核心？

Scheduler、quota、autoscaler 與 node-pressure ranking 都依賴 declared requests。它太高會浪費容量，太低會造成 overcommit 與 runtime pressure；應以 workload measurements、SLO 與 burst model 持續校準。

## 版本與內容新鮮度

本文件依 2026-08-21 的 Kubernetes 官方文件整理。已排除 deprecated kubelet dead-container GC flags、舊式 Scheduling Policy/Predicates/Priorities 與 `kubescheduler.config.k8s.io/v1alpha1` configuration 範例。Alpha scheduling features 已移至獨立延伸文件；部署前仍需以實際 cluster version 與 provider feature support 為準。

## 參考資料

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Resource Bin Packing](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)

---

# Kubernetes Scheduling, Preemption, and Eviction Interview Guide

Kubernetes scheduling is not merely finding a node with spare CPU. It translates a Pod's declared intent, cluster policy, resource availability, and disruption cost into a placement decision. `kube-scheduler` only selects a node for an unbound Pod. The kubelet enforces resources and performs node-pressure eviction, while voluntary disruption normally coordinates PodDisruptionBudget (PDB) through the Eviction API.

## Start with the complete lifecycle

```text
Pod admitted
  -> schedulingGates cleared when prerequisites are ready
  -> active scheduling queue, ordered by priority and QueueSort
  -> Filter: remove infeasible nodes
  -> PostFilter: optionally evaluate preemption
  -> Score: rank feasible nodes
  -> Reserve / Permit / PreBind
  -> Bind: write spec.nodeName through the API
  -> kubelet starts and enforces the Pod
  -> later disruption may come from preemption, node pressure, taints, or Eviction API
```

| Question | Decision maker | Key fact |
| --- | --- | --- |
| Where should a new Pod run? | `kube-scheduler` | Primarily uses requests, constraints, and cached cluster state, not live utilization |
| Has a container exceeded its runtime limit? | kubelet, container runtime, kernel | The scheduler does not enforce runtime isolation |
| Should a high-priority Pod remove a lower-priority Pod? | scheduler preemption | PDB is best effort; preemption does not create capacity |
| Which Pod should be removed under node pressure? | kubelet | Does not honor PDB; a hard threshold uses a `0s` grace period |
| Can planned drain remove a Pod? | API server Eviction subresource | Honors PDB and may return `429` |
| Should a `NoExecute` taint remove a Pod? | node lifecycle/taint manager and kubelet | Depends on toleration and `tolerationSeconds` |

## Scheduler pipeline and Scheduling Framework

The essential flow is **Filter -> Score -> Bind**:

- Filter decides whether a node is feasible based on resources, node affinity, taints, volume topology, ports, and other constraints.
- If no node is feasible, the Pod stays Pending. A PostFilter plugin can evaluate preemption, but removing Pods cannot solve every unschedulable cause.
- Score ranks only the nodes that passed Filter. The highest-scoring node wins, and the scheduler chooses between tied nodes.
- The scheduling cycle is serial and binding cycles can run concurrently. Extension points such as `Reserve`, `Permit`, and `PreBind` let plugins participate and roll back failed decisions.

Scheduling Framework profiles and plugins are the current extension model. Do not adopt legacy Scheduling Policies, Predicates/Priorities, or `kubescheduler.config.k8s.io/v1alpha1` examples; the current configuration API is `kubescheduler.config.k8s.io/v1`.

## Requests, Pod Overhead, and Scheduling Readiness

The scheduler uses requests rather than live metrics to assess capacity. Oversized CPU or memory requests create unnecessary Pending Pods and scaling; undersized requests overcommit nodes and increase throttling, OOM, and eviction risk.

For a sandboxed runtime, admission adds `RuntimeClass.overhead.podFixed` to `.spec.overhead`. This overhead counts toward ResourceQuota, scheduling, Pod cgroup sizing, and eviction ranking. Measure the runtime overhead instead of copying sample values.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: sandboxed
handler: sandboxed
overhead:
  podFixed:
    cpu: 250m
    memory: 120Mi
```

If a Pod should not be scheduled until a device claim, network attachment, or external controller is ready, initialize `.spec.schedulingGates` at creation. Gates can only be added at create time and can only be removed afterward. Clearing them moves the Pod toward the active scheduling queue. This avoids wasted scheduler and node-autoscaler work, but the gate owner needs timeouts, reconciliation, and observability to prevent permanent `SchedulingGated` Pods.

## Node selection from simple to complex

Use the smallest necessary constraint:

1. `nodeSelector`: all labels must match; suitable for a simple hard requirement.
2. Node affinity: `requiredDuringSchedulingIgnoredDuringExecution` is a Filter constraint; `preferredDuringSchedulingIgnoredDuringExecution` affects only Score.
3. Inter-Pod affinity/anti-affinity: uses other Pods' labels and topology placement to express co-location or separation.
4. `.spec.nodeName`: bypasses the scheduler and should be reserved for controlled cases; the Pod still fails if the node is absent or its kubelet rejects it.

`IgnoredDuringExecution` means a later node-label change does not automatically evict a bound Pod. Multiple selector and affinity conditions commonly form an AND, so excessive hard constraints can leave a Pod Pending forever. Inter-Pod affinity and anti-affinity also cost more to evaluate in large clusters and require consistent `topologyKey` labels on nodes.

Do not let an untrusted kubelet add an isolation label to itself. For security or isolation guarantees, use labels under the `node-restriction.kubernetes.io/` namespace protected by the NodeRestriction admission plugin and apply them through a trusted administrative path.

## Topology spread and availability

Pod topology spread constraints use `maxSkew` to control the difference in matching Pods across eligible topology domains. Common `topologyKey` values are `topology.kubernetes.io/zone` and `kubernetes.io/hostname`.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    minDomains: 3
    labelSelector:
      matchLabels:
        app: api
```

- `DoNotSchedule` is a hard constraint; `ScheduleAnyway` is a soft preference.
- `minDomains` is valid only with `DoNotSchedule`. If fewer eligible domains exist, the global minimum is treated as zero and the Pod may remain Pending.
- The `labelSelector` should match the workload template labels, or the new Pod may not contribute to skew calculations.
- Spread improves placement but is not HA by itself. You still need enough replicas, multi-zone capacity, PDB, readiness, and storage/network failure-domain design.
- Pod anti-affinity is useful for a "do not co-locate" rule; topology spread better expresses allowed skew across several domains.

## Taints and tolerations

A taint repels Pods. A toleration only permits placement; it does not attract a Pod to that node. A dedicated pool normally combines taint/toleration with node affinity so that the dedicated workload also avoids general nodes.

| Effect | New Pod | Running Pod |
| --- | --- | --- |
| `NoSchedule` | Rejected unless tolerated | Not actively evicted |
| `PreferNoSchedule` | Avoided when possible | Not actively evicted |
| `NoExecute` | Rejected unless tolerated | Evicted if not tolerated; `tolerationSeconds` can delay it |

Kubernetes maps node conditions to taints and gives ordinary Pods finite `not-ready` and `unreachable` tolerations by default. Avoid unlimited broad tolerations across all workloads because they undermine isolation and failure recovery.

## DRA and resource bin packing

The core of Dynamic Resource Allocation (DRA) is stable as of v1.35. A Pod can express device requirements through ResourceClaim or ResourceClaimTemplate, and the scheduler evaluates claim allocation, node placement, and device availability together. DRA is not equivalent to CPU or memory requests. DeviceClass, ResourceSlice, drivers, RBAC, and admin access have separate trust boundaries. Non-GA extensions are outside this baseline; see [Kubernetes Security Preview Features](security-preview-features.md) for security-related details.

The `NodeResourcesFit` `MostAllocated` or `RequestedToCapacityRatio` strategy can bin-pack according to the **requested-to-capacity ratio**, making low-utilization nodes easier to consolidate. It does not pack from live usage, so inaccurate requests still produce inaccurate placement.

Production tuning must account for:

- workload bursts and system/kube reserved headroom;
- zone, volume, and GPU/device fragmentation;
- autoscaler consolidation and replacement latency;
- failure blast radius and whether a node loss still meets the SLO.

## Priority and preemption

`PriorityClass` is cluster-scoped. Higher priority moves a Pending Pod earlier in the queue. If no node is feasible, default preemption may remove lower-priority Pods so that the preemptor can fit on one node.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: important-nonpreempting
value: 100000
globalDefault: false
preemptionPolicy: Never
description: Queue ahead of lower-priority work without evicting it.
```

- `preemptionPolicy: Never` keeps queue priority without actively preempting; a still-higher-priority Pod can preempt it.
- Preemption removes only the necessary lower-priority victims, and capacity is not guaranteed to remain available to the original preemptor while victims terminate.
- PDB support in scheduler preemption is best effort. Preemption can still violate a PDB if no compliant victim set exists.
- Removing low-priority Pods may not help if the blocker is node affinity, a taint, volume topology, a port conflict, or a missing device.
- Govern PriorityClass use through admission and RBAC. If everything is high priority, priority becomes meaningless and can cause cascading disruption.

## Do not confuse the three eviction paths

| Mechanism | Trigger | PDB | Grace period | Typical purpose |
| --- | --- | --- | --- | --- |
| Scheduler preemption | `kube-scheduler` | Best effort | Uses victim termination grace | Make room for a high-priority Pending Pod |
| Node-pressure eviction | kubelet | Not honored | `0s` for hard thresholds; soft thresholds use kubelet maximum grace | Protect the node from memory, disk, or PID starvation |
| API-initiated eviction | drain, autoscaler, or API client | Honored | Normal Pod termination flow | Planned maintenance and voluntary disruption |

The Eviction API creates a `policy/v1` Eviction subresource. An allowed eviction returns `200`; a temporary PDB denial normally returns `429`. Directly deleting a Pod bypasses PDB, so maintenance automation should prefer the Eviction API, handle retry and backoff, and assess application risk before force deletion.

Node-pressure eviction first attempts node-level reclaim, such as removing dead containers or unused images. It then ranks victims by whether usage exceeds requests, Pod Priority, and usage relative to requests. This is not a simple QoS ordering, and disk or PID pressure has no equivalent request-based model. Monitor `memory.available`, `nodefs`/`imagefs`/`containerfs` availability and inodes, and `pid.available`. Configure grace for soft thresholds and sufficient safety margin for hard thresholds.

Do not use deprecated kubelet dead-container garbage-collection flags. Use current eviction thresholds and runtime/image garbage collection mechanisms.

## Scheduler performance tuning

`percentageOfNodesToScore` lets the scheduler stop searching after finding enough feasible nodes, trading placement accuracy for scheduling latency in large clusters. The default is calculated from cluster size. Clusters with a few hundred nodes or fewer should normally retain it, and values below 10% can frequently miss better placements.

For any tuning, first prove the bottleneck with scheduler metrics or profiling, then compare scheduling latency, attempts, unschedulable Pods, plugin duration, and placement quality. Managed Kubernetes often does not expose control-plane scheduler configuration, so check the provider boundary first.

## Preview scheduling features

PodGroup, Gang Scheduling, Topology-Aware Scheduling, Workload-Aware Preemption, and other immature scheduler capabilities are outside this baseline. See [Kubernetes Alpha Scheduling Features](scheduling-alpha-features.md) for group scheduling and [Kubernetes Beta Scheduling Features](scheduling-beta-features.md) for scheduler optimization.

## Troubleshooting Pending Pods and evictions

1. Use `kubectl describe pod` to inspect the `PodScheduled` condition, events, `SchedulingGated`, and nominated node.
2. Verify that requests and overhead fit node allocatable; do not rely only on live `kubectl top` output.
3. Check nodeSelector/affinity, taints/tolerations, topology spread, ports, PVC/volume topology, and DRA claims one at a time.
4. Check scheduler queue, latency, plugin metrics/logs, and whether a webhook or controller still owns a gate.
5. For preemption, inspect PriorityClass, victims, PDB, and termination time. A nominated node does not guarantee a successful bind.
6. For eviction, first identify kubelet pressure, Eviction API, taint, or scheduler preemption, then inspect the corresponding events, node conditions, and audit records.
7. Change hard constraints, priority, or eviction thresholds only after validating with dry runs, staging load, and a rollback plan.

## Common interview questions

### How do toleration and node affinity differ?

A toleration permits a Pod onto a tainted node but does not attract it there. Node affinity selects nodes by labels. A dedicated pool commonly needs both.

### How do you choose between topology spread and Pod anti-affinity?

Anti-affinity expresses that Pods should not be co-located, and hard anti-affinity commonly permits only one matching Pod per domain. Topology spread directly controls skew across several domains and is usually more expressive for availability across many replicas.

### Can PDB prevent every eviction?

No. The Eviction API honors PDB, scheduler preemption treats it as best effort, node-pressure eviction does not honor it, and direct Pod deletion bypasses it.

### Why are requests central to scheduling reliability?

The scheduler, quota, autoscaler, and node-pressure ranking depend on declared requests. Oversizing wastes capacity; undersizing causes overcommit and runtime pressure. Calibrate requests continuously from workload measurements, SLOs, and a burst model.

## Version and freshness notes

This guide reflects the official Kubernetes documentation reviewed on 2026-08-21. It excludes deprecated kubelet dead-container GC flags, legacy Scheduling Policy/Predicates/Priorities, and `kubescheduler.config.k8s.io/v1alpha1` configuration examples. Alpha scheduling features have moved to a separate supplementary guide. Always verify the actual cluster version and provider feature support before deployment.

## References

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Resource Bin Packing](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
