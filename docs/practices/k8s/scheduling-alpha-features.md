# Kubernetes Alpha Scheduling Features 延伸指南

本文件隔離 Kubernetes v1.35–v1.36 的 alpha scheduling capabilities，避免將實驗性 API 與 [stable scheduling baseline](scheduling.md) 混在一起。這些功能適合用來理解 AI/ML、HPC 與 distributed batch workloads 的設計方向，不應因為出現在官方文件就直接視為 production-ready。

## 為何 group scheduling 不同於一般 Pod scheduling

一般 `kube-scheduler` 逐一處理 Pods。若一個 distributed training job 需要八個 workers 才能開始，兩個 jobs 各自先取得四個 Pods，可能耗盡 capacity，卻沒有任何 job 能做有效工作。Group scheduling 嘗試把多個 Pods 視為共同的 scheduling unit，避免 partial placement 與 resource deadlock。

```text
Workload / PodGroup
  -> collect enough Pods for the group
  -> generate candidate placements
  -> evaluate every Pod against resources and constraints
  -> satisfy gang minimum or reject the whole placement
  -> reserve and bind as one coordinated decision
```

## Feature maturity

| 功能 | 官方文件狀態 | 解決的問題 | 主要風險 |
| --- | --- | --- | --- |
| PodGroup scheduling | v1.35 alpha，預設關閉 | 以 group 為單位執行 scheduling cycle | API 與 algorithm 仍可能改變 |
| Gang Scheduling | v1.35 alpha，預設關閉 | 全部或至少 `minCount` 可行才 bind | queue starvation、capacity deadlock |
| Placement scheduling algorithm | v1.36 alpha，預設關閉 | 產生並評分 PodGroup candidate placements | plugin behavior 與 scaling cost |
| Topology-Aware Scheduling（TAS） | v1.36 alpha，預設關閉 | 把整個 PodGroup 放在同一 topology domain | domain capacity fragmentation |
| Workload-Aware Preemption | v1.36 alpha，預設關閉 | 以 PodGroup 而非單一 Pod 選擇 preemptor/victims | cluster-wide disruption blast radius |
| Numeric toleration operators | v1.35 alpha，預設關閉 | 以 `Gt`/`Lt` 比較 numeric taint values | portability 與 validation risk |

Alpha feature 預設關閉，API 可能無 backward-compatibility 保證，也可能在未來版本被大幅修改或移除。Managed Kubernetes provider 可能不允許啟用相關 API group、feature gate 或 scheduler plugin。

## PodGroup scheduling cycle

PodGroup scheduling 依賴 Workload API、`GenericWorkload` feature gate 與 alpha `scheduling.k8s.io` API。Scheduler 會在同一 cycle 內評估 group 的多個 Pods，暫時假設與 reserve 可行 placement；若整組條件未滿足，就不進入 binding。

目前 default algorithm 仍有重要限制：Pod 處理順序可能讓 scheduler 錯過實際存在的可行 placement。Homogeneous Pods、沒有複雜 inter-Pod dependency 的 group 較容易推理；heterogeneous requests、affinity、topology spread 與 device constraints 會增加失敗情境。

## Gang Scheduling

Gang Scheduling 以 `minCount` 表達最低共同啟動數量。Pods 在 referenced PodGroup 存在且已建立的 Pods 達到 `minCount` 前，不會進入 active scheduling queue；達到 quorum 後才嘗試 coordinated placement。

它能避免 partial startup，但不會自動創造 capacity。若多個 gangs 同時競爭，仍需 workload queueing、fairness、admission control、timeout 與 capacity planning，否則大型 gang 可能長期等待或阻塞較小 workloads。

## Placement 與 Topology-Aware Scheduling

Placement scheduling algorithm 先產生 candidate placements，再逐一驗證 group 中的 Pods，最後由 placement score plugins 選出最佳方案。TAS 使用 topology key 將 nodes 分成 domains，並要求整個 PodGroup co-locate 在同一 domain；`NodeResourcesFit` 可用 `MostAllocated` 對 placement 做 bin-packing score。

TAS 與 stable Pod topology spread constraints 不相同：

- Stable topology spread 逐 Pod 控制 replicas 跨 domains 的 skew，常用於 availability。
- Alpha TAS 為 PodGroup 選擇單一共同 domain，常用於降低 distributed workload 的 network latency 或符合 accelerator topology。

採用前要驗證單一 domain 是否有完整 CPU、memory、device、volume 與 network capacity；co-location 也會擴大 domain failure 的影響。

## Workload-Aware Preemption

Workload-Aware Preemption 只用於 PodGroup scheduling。它把整個 PodGroup 當成 preemptor，在 cluster-wide domain 尋找 victims，並能依另一個 PodGroup 的 disruption mode 將其視為完整 victim unit。這與 stable default preemption 逐 node、逐 Pod 尋找 lower-priority victims 的模型不同。

啟用前必須明確回答：

- 哪些 workloads 允許整組被 preempt，checkpoint/restart cost 是多少？
- Priority 與 workload queue policy 是否會 starvation？
- PDB、termination time 與 replacement controller 如何交互作用？
- 一次跨 nodes 移除多個 victims 是否仍符合服務 SLO？

## Numeric toleration operators

Alpha `Gt` 與 `Lt` operators 可將 taint 與 toleration value 當作 signed 64-bit integer 比較，適用於 reliability tier 或 threshold 類 placement。它們不是一般 taint/toleration 的必要能力；production baseline 應繼續使用 stable `Equal` 與 `Exists`，除非有明確需求並完成 version、admission validation 與 rollback 測試。

## 採用 gate

只有在下列條件都具備時才考慮實驗：

1. 實際 cluster 與 provider 明確支援所需 feature gates、API groups 與 scheduler configuration。
2. 有 non-production cluster、representative load、failure injection 與 rollback path。
3. 已測試 queue fairness、partial failure、scheduler restart、control-plane upgrade 與 mixed-version behavior。
4. 有 Pending/Gated duration、group scheduling latency、victim count、preemption rate 與 useful-work ratio 的 observability。
5. 已比較成熟 batch scheduler ecosystem，並能說明選擇 in-tree alpha feature 的理由。

## 面試回答邊界

面試中應先說明這些是 alpha、預設關閉，再解釋它們解決的 partial placement 問題。不要宣稱 Kubernetes 已有普遍穩定的 native gang scheduling；應補充 provider support、API evolution、fairness、preemption blast radius 與 rollback trade-offs。

## 版本與內容新鮮度

本文件依 2026-08-21 的 Kubernetes v1.36 官方文件整理，只追蹤 alpha scheduling features。它不是 deployment runbook；feature maturity、API version、default state 與 managed-service support 都必須在實驗前重新確認。

## 參考資料

- [Topology-Aware Workload Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
- [PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [Workload-Aware Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/)

---

# Kubernetes Alpha Scheduling Features Supplement

This guide isolates alpha scheduling capabilities from Kubernetes v1.35-v1.36 so that experimental APIs do not get mixed into the [stable scheduling baseline](scheduling.md). These features are useful for understanding the design direction for AI/ML, HPC, and distributed batch workloads. Their presence in the official documentation does not make them production-ready.

## Why group scheduling differs from ordinary Pod scheduling

The standard `kube-scheduler` processes Pods individually. If a distributed training job needs eight workers to start, two jobs might each acquire four Pods, exhaust capacity, and still perform no useful work. Group scheduling tries to treat several Pods as one scheduling unit to avoid partial placement and resource deadlock.

```text
Workload / PodGroup
  -> collect enough Pods for the group
  -> generate candidate placements
  -> evaluate every Pod against resources and constraints
  -> satisfy gang minimum or reject the whole placement
  -> reserve and bind as one coordinated decision
```

## Feature maturity

| Feature | Official documentation state | Problem addressed | Main risk |
| --- | --- | --- | --- |
| PodGroup scheduling | v1.35 alpha, disabled by default | Run a scheduling cycle for a group | API and algorithm can still change |
| Gang Scheduling | v1.35 alpha, disabled by default | Bind only when all Pods or at least `minCount` are feasible | Queue starvation and capacity deadlock |
| Placement scheduling algorithm | v1.36 alpha, disabled by default | Generate and score PodGroup candidate placements | Plugin behavior and scaling cost |
| Topology-Aware Scheduling (TAS) | v1.36 alpha, disabled by default | Place a complete PodGroup in one topology domain | Domain capacity fragmentation |
| Workload-Aware Preemption | v1.36 alpha, disabled by default | Select preemptors and victims as PodGroups rather than Pods | Cluster-wide disruption blast radius |
| Numeric toleration operators | v1.35 alpha, disabled by default | Compare numeric taint values with `Gt` or `Lt` | Portability and validation risk |

Alpha features are disabled by default. Their APIs might not offer backward-compatibility guarantees and can be substantially changed or removed in future versions. A managed Kubernetes provider might not expose the required API group, feature gate, or scheduler plugin.

## PodGroup scheduling cycle

PodGroup scheduling depends on the Workload API, the `GenericWorkload` feature gate, and an alpha `scheduling.k8s.io` API. The scheduler evaluates several group Pods in one cycle and tentatively assumes and reserves feasible placements. It does not enter binding if the group criteria are not met.

The current default algorithm has an important limitation: Pod processing order can cause it to miss a feasible placement that actually exists. Homogeneous Pods without complex inter-Pod dependencies are easier to reason about. Heterogeneous requests, affinity, topology spread, and device constraints add failure cases.

## Gang Scheduling

Gang Scheduling expresses a minimum coordinated startup size through `minCount`. Pods do not enter the active scheduling queue until the referenced PodGroup exists and the number of created Pods reaches `minCount`. Only after that quorum does the scheduler attempt coordinated placement.

This avoids partial startup but does not create capacity. When multiple gangs compete, workload queueing, fairness, admission control, timeouts, and capacity planning are still required. Otherwise, a large gang can wait indefinitely or block smaller workloads.

## Placement and Topology-Aware Scheduling

The placement scheduling algorithm first generates candidate placements, validates the group Pods against each one, and uses placement score plugins to choose the best result. TAS groups nodes into domains by topology key and requires the complete PodGroup to be co-located in one domain. `NodeResourcesFit` can use `MostAllocated` to bin-pack a placement.

TAS differs from stable Pod topology spread constraints:

- Stable topology spread controls per-Pod replica skew across domains and commonly supports availability.
- Alpha TAS chooses one common domain for a PodGroup, commonly to reduce distributed-workload network latency or meet accelerator topology.

Before adoption, verify that one domain has complete CPU, memory, device, volume, and network capacity. Co-location also increases the impact of a domain failure.

## Workload-Aware Preemption

Workload-Aware Preemption applies only to PodGroup scheduling. It treats the entire PodGroup as a preemptor, searches for victims across a cluster-wide domain, and can use another PodGroup's disruption mode to treat it as a complete victim unit. This differs from stable default preemption, which searches per node for lower-priority Pod victims.

Before enabling it, answer these questions explicitly:

- Which workloads may be preempted as a group, and what is their checkpoint or restart cost?
- Can Priority and workload queue policy cause starvation?
- How do PDB, termination time, and replacement controllers interact?
- Does removing several victims across nodes still satisfy service SLOs?

## Numeric toleration operators

The alpha `Gt` and `Lt` operators compare taint and toleration values as signed 64-bit integers, which can express reliability tiers or threshold-based placement. They are not necessary for ordinary taints and tolerations. A production baseline should continue to use stable `Equal` and `Exists` unless a specific need justifies version, admission-validation, and rollback testing.

## Adoption gate

Consider an experiment only when all of these conditions hold:

1. The actual cluster and provider explicitly support the required feature gates, API groups, and scheduler configuration.
2. A non-production cluster, representative load, failure injection, and rollback path are available.
3. Queue fairness, partial failure, scheduler restart, control-plane upgrade, and mixed-version behavior have been tested.
4. Observability covers Pending or Gated duration, group scheduling latency, victim count, preemption rate, and useful-work ratio.
5. A mature batch-scheduler ecosystem has been compared, and the reason for selecting an in-tree alpha feature is defensible.

## Interview answer boundary

In an interview, first state that these features are alpha and disabled by default, then explain the partial-placement problem they address. Do not claim that Kubernetes already offers universally stable native gang scheduling. Include provider support, API evolution, fairness, preemption blast radius, and rollback trade-offs.

## Version and freshness notes

This guide reflects the Kubernetes v1.36 documentation reviewed on 2026-08-21 and tracks alpha scheduling features only. It is not a deployment runbook. Recheck feature maturity, API version, default state, and managed-service support before every experiment.

## References

- [Topology-Aware Workload Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
- [PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [Workload-Aware Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/)
