# Kubernetes Policy：資源治理與 PID 防護面試指南

Kubernetes policy 的核心不是「設定一個上限」而已，而是分清楚 policy 在哪一層生效：`LimitRange` 約束 namespace 內的單一 Pod、Container 或 PVC，`ResourceQuota` 限制 namespace 的總消耗與物件數量，而 PID limit/reservation 在 node 的 kubelet 與 operating system 層防止 process exhaustion。三者互補，不能互相取代。

## 先掌握 enforcement path

```text
API request
  -> LimitRanger admission: default and validate each object
  -> ResourceQuota admission: check namespace aggregate usage
  -> object stored
  -> controller creates Pods
  -> scheduler places Pods according to requests
  -> kubelet/runtime enforces CPU, memory and Pod PID limits
  -> node-pressure eviction observes signals such as pid.available
```

- Admission rejection 通常回傳 `403 Forbidden`；這代表 desired object 不符合 policy，不代表 scheduler 或 runtime failure。
- `LimitRange` 與 `ResourceQuota` 主要在 create/update admission 時檢查，新建或修改 policy 不會追溯改寫既有物件。
- Deployment 本身可能成功建立，但其 ReplicaSet 建立 Pod 時被 quota 或 limit policy 拒絕；因此必須檢查 workload conditions、events 與 ReplicaSet，而不能只看 `kubectl apply` 成功。
- Admission policy 管理 declared resources；實際 resource isolation、scheduling 與 node stability 仍依賴 scheduler、kubelet、cgroups、container runtime 與 OS。

## 三種機制的責任邊界

| 機制 | Scope | 生效時間 | 解決的問題 | 不保證什麼 |
| --- | --- | --- | --- | --- |
| `LimitRange` | Namespace 內的單一 Pod、Container 或 PVC | Admission | default、minimum、maximum、request-to-limit ratio | Namespace 總量與 cluster capacity |
| `ResourceQuota` | Namespace aggregate | Admission + usage accounting | CPU、memory、storage、extended resources、object count 的總上限 | Capacity reservation、公平排程或單一物件大小 |
| PID reservation | Node | Kubelet/node configuration | 為 OS 與 Kubernetes daemons 保留 PIDs | 單一 Pod 不濫用 PIDs |
| Pod PID limit | 每個 Node 上的每個 Pod | Runtime | 限制 fork bomb 或異常 process growth 的 blast radius | Scheduler capacity accounting 或跨 Nodes 一致性 |
| `pid.available` eviction | Node | Periodic runtime observation | Node PID pressure 時驅逐 Pods | 即時 hard limit；快速耗盡仍可能讓 node 不穩定 |

## LimitRange：單一物件的 guardrail

`LimitRange` 可設定 Container/Pod compute resource 的 minimum、maximum 與 request/limit ratio，也能為 Container 注入 default request/default limit，並約束 PVC 的 storage request。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: workload-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 2Gi
      maxLimitRequestRatio:
        cpu: "10"
        memory: "4"
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 100Gi
```

關鍵細節：

- Defaulting 發生在 admission；應以 server-side dry run 或讀回 object 驗證最終 resources，而不是只檢查提交前的 YAML。
- `default` 是 limit，`defaultRequest` 是 request。若只設定 limit，Kubernetes 的 resource defaulting rules 可能使 request 等於 limit，影響 QoS、bin packing 與可排程容量。
- LimitRange 不會驗證它注入的 default 是否與使用者提供的另一個值一致。例如 default limit 小於明確 request 時，最終 Pod 可能無法被接受或排程；policy 必須以 representative workloads 測試。
- 同一 namespace 有多個 LimitRanges 時，套用哪個 default 並不 deterministic。Production 應避免重疊 default policy，並建立單一 ownership 與 GitOps source of truth。
- Policy 只影響後續 admission；變更後應盤點並逐步重建既有 workloads，避免新舊 Pods 使用不同 resources。

## ResourceQuota：namespace 的總預算

`ResourceQuota` 可限制 non-terminal Pods 的 aggregate requests/limits、PVC/storage、extended resources，以及 namespaced API objects 的數量。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-budget
  namespace: team-a
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
    count/pods: "100"
    count/services: "20"
    services.loadbalancers: "2"
    count/secrets: "100"
```

- CPU/memory quota 啟用後，未宣告相關 request 或 limit 的新 Pod 可能被拒絕；通常搭配 `LimitRange` defaulting，讓 quota accounting 有完整輸入。
- Object-count quota 不只控制成本，也保護 API server/etcd、Pod IP、LoadBalancer、NodePort 與 controller capacity。對 CRD 可用 `count/<plural>.<api-group>`；aggregated API 是否 enforcement 由 extension API server 負責。
- Storage 可限制總 `requests.storage`、PVC count，或以 `<storage-class>.storageclass.storage.k8s.io/...` 分類，避免昂貴 tier 被無限制使用。
- Extended resources（例如 GPU）通常 quota `requests.<resource-name>`；不要假設 quota 代表 device 已被保留或一定可排程。
- Quota 是 upper bound，不是 reservation。所有 namespaces quota 總和可以超過 cluster capacity；實際資源仍可能 first-come-first-served，需搭配 capacity planning、PriorityClass 與 autoscaling。
- 限制誰能 update/delete `ResourceQuota` 與 `LimitRange`；否則 tenant 可移除自己的 guardrail。使用 RBAC，必要時再以 admission policy 保護 policy objects。

## Scope 與 PriorityClass

ResourceQuota 可用 `scopes` 或 `scopeSelector` 只計算符合條件的 workload，例如 `BestEffort`、`NotBestEffort`、`Terminating`、`NotTerminating`、`PriorityClass` 與 `CrossNamespacePodAffinity`。

- PriorityClass-scoped quota 適合為 platform-critical 與 tenant workloads 建立不同 budget，但必須同時嚴格限制誰能使用高 priority；否則 quota segmentation 可能成為 privilege escalation 或 starvation 路徑。
- `CrossNamespacePodAffinity` scope 可限制會跨 namespace 影響 scheduling domain 的 affinity/anti-affinity，降低單一 tenant 阻擋其他 workloads 的風險。
- Scope 不是 label selector 的通用替代品；只有 API 支援的 scope names/operators/resources 可用。部署前應以目標 Kubernetes version 的 API schema 驗證。

## PID limits 與 reservations

PID exhaustion 可能阻止 node 建立新 processes，連 kubelet、runtime、SSH 或監控 agent 都可能受影響。PID protection 必須同時處理 system reserve、per-Pod hard boundary 與 node-pressure response。

```yaml
# KubeletConfiguration fragment; validate apiVersion against the target cluster.
podPidsLimit: 4096
systemReserved:
  pid: "1000"
kubeReserved:
  pid: "1000"
evictionHard:
  pid.available: "10%"
```

- `podPidsLimit` 是 node-level kubelet setting，但限制套用到該 node 上的每個 Pod；Kubernetes 目前沒有可由 Pod spec 宣告並由 scheduler accounting 的 PID resource。
- 所有 worker nodes 宜使用一致設定，否則 Pod reschedule 後的 PID ceiling 會改變。異質 node pool 若刻意不同，應用 labels/taints、documented policy 與 conformance checks 管理。
- `systemReserved.pid` 與 `kubeReserved.pid` 保留 node processes 給 OS 和 Kubernetes daemons；它們不是 Pod limit。
- `pid.available` eviction 是定期觀察後採取反應，不是 hard enforcement。高速 fork bomb 可能在 eviction loop 反應前耗盡 PIDs，因此不能取代 `podPidsLimit`。
- Production 優先使用 version-controlled kubelet configuration，而不是只依賴 command-line `--pod-max-pids`，並確認 managed Kubernetes provider 是否允許及如何 rollout node configuration。

## 與 requests、QoS、autoscaling 的互動

- CPU request 影響 scheduling，CPU limit 通常由 throttling enforcement；memory limit 可能導致 OOM kill。Quota 只加總 declared values，不代表真實 usage 或效能需求。
- LimitRange default 方便 onboarding，但錯誤 default 會系統性造成 over-request、CPU throttling、OOM 或低 node utilization。Default 應來自量測、load test 與 workload class，而不是任意統一值。
- HPA 的 CPU utilization 通常以 usage/request 為分母；default 或調整 request 會改變 scaling signal。修改 policy 前須模擬 HPA thresholds 與 replica behavior。
- VPA recommendation、quota headroom 與 rollout strategy 必須一起評估；VPA 建議可因 quota 或 LimitRange max 而無法套用。
- PID 不屬於 standard Pod request，scheduler 不會因 PID headroom 自動做 bin packing；需監控 node/process pressure 並保留容量。

## Production rollout checklist

1. 以 historical usage、SLO、burst、replica limits、storage growth 與 failure scenarios 建立 namespace budget。
2. 先在 audit/test namespace 驗證 representative Deployments、Jobs、CronJobs、sidecars、init containers 與 PVCs。
3. 使用 server-side dry run 與 policy tests 驗證 defaulted object、min/max、ratio、quota scopes 與 expected rejection cases。
4. 預留 rollout、HPA scale-out、Job retries、surge Pods、incident debugging 與 failover headroom；quota 不應等於平時 usage。
5. 監控 quota `used/hard`、admission rejections、pending/failed Pods、CPU throttling、OOM、PVC growth、Pod count 與 node `pid.available`。
6. 以 RBAC/admission 保護 policy objects；變更走 review、canary namespace、rollback 與 post-change validation。
7. Policy 變更不追溯既有 objects；定義 restart/recreate 計畫，並比較 rollout 前後實際 spec。

## Troubleshooting：判斷失敗發生在哪一層

1. `kubectl describe` workload、ReplicaSet、Pod、PVC，讀取 events 中的 `exceeded quota`、minimum/maximum、ratio 或 missing request/limit 訊息。
2. `kubectl get resourcequota,limitrange -n <namespace> -o yaml`，比較 quota `status.used`/`status.hard` 與 LimitRange default/constraints。
3. 若 Deployment 存在但 replicas 不足，檢查 ReplicaSet events；top-level object admission success 不代表 child Pods 被接受。
4. 若 Pod 已被接受但 `Pending`，檢查 requests、node allocatable、taints/topology 與 PVC；quota success 不代表 capacity available。
5. 若 process creation 出現 `resource temporarily unavailable`、node 不穩定或 `PIDPressure`，檢查 kubelet config、node PID usage、per-Pod process count 與 eviction events。
6. 修正 policy 前確認 ownership；不要只提高 quota 隱藏 resource leak、runaway CronJob 或 fork bomb。

## 常見面試題的精準回答

### LimitRange 和 ResourceQuota 有何差異？

LimitRange 約束並可 default 單一 object 的 resource declaration；ResourceQuota 限制 namespace 的 aggregate consumption/object count。通常先由 LimitRange 補齊或驗證 requests/limits，再由 ResourceQuota 檢查加入後是否超過總預算。

### ResourceQuota 是否保證 namespace 有那些資源？

不保證。Quota 是 consumption ceiling，不是 capacity reservation。Namespace 即使仍有 quota headroom，也可能因 cluster capacity、topology、taints、storage、attach limit 或 priority 而無法排程。

### 為何 Deployment 建立成功但沒有 Pods？

Admission 是逐物件執行。Deployment 可先被儲存，之後 ReplicaSet 建立 Pod 時才觸發 LimitRange/ResourceQuota；若 Pod 被拒絕，應在 ReplicaSet 或 Deployment events 看到原因。

### `pid.available` eviction 能阻止 fork bomb 嗎？

不能單獨保證。Eviction 依週期性 signal 反應，PID 可能在下一次檢查前快速耗盡。應以 per-Pod `podPidsLimit` 建立 hard boundary，再搭配 reservations、eviction、monitoring 與 runtime hardening。

### 為何不應建立多個重疊的 LimitRanges？

同 namespace 多個 LimitRanges 若都提供 defaults，哪個 default 被採用並不 deterministic，容易讓 admission 結果與團隊預期不同。Production 應有單一 ownership，避免重疊 defaulting，並測試最終 object。

## 版本與過時內容判讀

- 這三份 Kubernetes v1.36 文件中的 `LimitRange`、`ResourceQuota`、PID reservation、`podPidsLimit` 與 `pid.available` 都是現行機制，沒有需要沿用的已移除 API object。
- 官方 PID 文件仍列出 kubelet `--pod-max-pids` command-line option；本指南保留其概念但以 version-controlled kubelet configuration 為 production default。Managed service 的支援與欄位 rollout 方式必須查供應商/版本文件。
- Local ephemeral-storage quota 仍非 GA，已移至 [Kubernetes Policy Preview Features](policy-preview-features.md)，不列入本 production baseline。
- 新增或 version-sensitive quota scopes/resources 應先對照目標 cluster 版本；不要把最新網站的 v1.36 範例直接套用到較舊 cluster。

## 官方參考資料

- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Process ID Limits and Reservations](https://kubernetes.io/docs/concepts/policy/pid-limiting/)

---

# Kubernetes Policy: Interview Guide to Resource Governance and PID Protection

Kubernetes policy is not merely about setting a limit. The key is knowing where each policy is enforced: `LimitRange` constrains an individual Pod, container, or PVC in a namespace; `ResourceQuota` caps aggregate namespace consumption and object counts; and PID limits and reservations protect nodes at the kubelet and operating-system layer. These mechanisms complement rather than replace one another.

## Start with the enforcement path

```text
API request
  -> LimitRanger admission: default and validate each object
  -> ResourceQuota admission: check namespace aggregate usage
  -> object stored
  -> controller creates Pods
  -> scheduler places Pods according to requests
  -> kubelet/runtime enforces CPU, memory and Pod PID limits
  -> node-pressure eviction observes signals such as pid.available
```

- Admission rejection usually returns `403 Forbidden`; this means the desired object violates policy, not that scheduling or runtime failed.
- `LimitRange` and `ResourceQuota` primarily check create/update admission. Creating or changing policy does not retroactively rewrite existing objects.
- A Deployment can be admitted while its ReplicaSet is later unable to create Pods because of quota or limit policy. Inspect workload conditions, events, and the ReplicaSet rather than treating a successful `kubectl apply` as successful execution.
- Admission policy governs declared resources; actual isolation, scheduling, and node stability still depend on the scheduler, kubelet, cgroups, container runtime, and OS.

## Responsibility boundaries

| Mechanism | Scope | Enforcement time | Problem addressed | What it does not guarantee |
| --- | --- | --- | --- | --- |
| `LimitRange` | Individual Pod, container, or PVC in a namespace | Admission | Defaults, minimums, maximums, request-to-limit ratio | Namespace totals or cluster capacity |
| `ResourceQuota` | Namespace aggregate | Admission plus usage accounting | Aggregate CPU, memory, storage, extended resources, and object counts | Capacity reservation, fair scheduling, or individual object size |
| PID reservation | Node | Kubelet/node configuration | Reserves PIDs for the OS and Kubernetes daemons | That one Pod cannot abuse PIDs |
| Pod PID limit | Each Pod on each node | Runtime | Bounds the blast radius of fork bombs or abnormal process growth | Scheduler capacity accounting or consistency across nodes |
| `pid.available` eviction | Node | Periodic runtime observation | Evicts Pods during node PID pressure | An immediate hard limit; fast exhaustion can still destabilize a node |

## LimitRange: guardrails for individual objects

A `LimitRange` can define minimum, maximum, and request-to-limit ratios for Pod/container compute resources, inject default container requests and limits, and constrain PVC storage requests.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: workload-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 2Gi
      maxLimitRequestRatio:
        cpu: "10"
        memory: "4"
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 100Gi
```

Important details:

- Defaulting happens at admission. Use server-side dry run or read the object back to verify final resources rather than inspecting only the submitted manifest.
- `default` is the limit and `defaultRequest` is the request. If only a limit is set, Kubernetes resource defaulting rules can make the request equal to the limit, affecting QoS, bin packing, and schedulable capacity.
- LimitRange does not verify that an injected default is consistent with another user-provided value. For example, a default limit below an explicit request can produce an object that cannot be accepted or scheduled. Test policy against representative workloads.
- When a namespace has multiple LimitRanges, which default is applied is nondeterministic. Avoid overlapping default policies in production, and establish one owner and one GitOps source of truth.
- Policy affects later admissions only. Inventory and gradually recreate existing workloads after a change so that old and new Pods do not silently use different resources.

## ResourceQuota: the namespace budget

A `ResourceQuota` can limit aggregate requests and limits for non-terminal Pods, PVC/storage use, extended resources, and counts of namespaced API objects.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-budget
  namespace: team-a
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
    count/pods: "100"
    count/services: "20"
    services.loadbalancers: "2"
    count/secrets: "100"
```

- After CPU or memory quota is enabled, a new Pod that omits the relevant request or limit may be rejected. Pair quota with `LimitRange` defaulting so accounting receives complete inputs.
- Object-count quota controls cost and protects API server/etcd, Pod IP, LoadBalancer, NodePort, and controller capacity. Use `count/<plural>.<api-group>` for CRDs; an extension API server is responsible for enforcement on aggregated APIs.
- Storage can be limited by aggregate `requests.storage`, PVC count, or `<storage-class>.storageclass.storage.k8s.io/...`, preventing unrestricted use of an expensive tier.
- Extended resources such as GPUs are commonly quota-controlled through `requests.<resource-name>`. Quota does not mean a device is reserved or schedulable.
- Quota is an upper bound, not a reservation. The sum of namespace quotas may exceed cluster capacity; resources can still be first-come-first-served, requiring capacity planning, PriorityClass, and autoscaling.
- Restrict who can update or delete `ResourceQuota` and `LimitRange`, or tenants can remove their own guardrails. Use RBAC and, where needed, admission policy to protect policy objects.

## Scopes and PriorityClass

ResourceQuota can use `scopes` or `scopeSelector` to count only matching workloads, including `BestEffort`, `NotBestEffort`, `Terminating`, `NotTerminating`, `PriorityClass`, and `CrossNamespacePodAffinity`.

- PriorityClass-scoped quota can separate budgets for platform-critical and tenant workloads, but access to high priority must also be tightly restricted; otherwise quota segmentation can become a privilege-escalation or starvation path.
- The `CrossNamespacePodAffinity` scope can limit affinity/anti-affinity that affects scheduling domains across namespaces, reducing one tenant's ability to block other workloads.
- Scope is not a general-purpose label selector. Only supported scope names, operators, and resources are valid; verify the target Kubernetes version's API schema before deployment.

## PID limits and reservations

PID exhaustion can prevent a node from creating processes, affecting kubelet, the runtime, SSH, and monitoring agents. PID protection needs system reservation, a per-Pod hard boundary, and a node-pressure response.

```yaml
# KubeletConfiguration fragment; validate apiVersion against the target cluster.
podPidsLimit: 4096
systemReserved:
  pid: "1000"
kubeReserved:
  pid: "1000"
evictionHard:
  pid.available: "10%"
```

- `podPidsLimit` is a node-level kubelet setting whose limit applies to each Pod on that node. Kubernetes currently has no Pod-spec PID resource that the scheduler accounts for.
- Keep the setting consistent across workers so rescheduling does not change a Pod's PID ceiling. If heterogeneous node pools are intentional, govern them with labels/taints, documented policy, and conformance checks.
- `systemReserved.pid` and `kubeReserved.pid` preserve node processes for the OS and Kubernetes daemons; they are not Pod limits.
- `pid.available` eviction reacts after periodic observation and is not hard enforcement. A fast fork bomb may exhaust PIDs before the eviction loop responds, so it cannot replace `podPidsLimit`.
- Prefer version-controlled kubelet configuration in production over relying only on `--pod-max-pids`, and verify how the managed Kubernetes provider supports and rolls out node configuration.

## Interaction with requests, QoS, and autoscaling

- CPU requests affect scheduling and CPU limits are generally enforced through throttling; memory limits can cause OOM kills. Quota sums declared values, not actual usage or performance demand.
- LimitRange defaults ease onboarding, but bad defaults systematically cause over-requesting, CPU throttling, OOMs, or poor utilization. Derive defaults from measurements, load tests, and workload classes.
- HPA CPU utilization commonly uses usage divided by request; defaulting or changing requests changes the scaling signal. Model HPA thresholds and replica behavior before changing policy.
- Evaluate VPA recommendations, quota headroom, and rollout strategy together; a recommendation may be impossible to apply because of quota or a LimitRange maximum.
- PID is not a standard Pod request, so the scheduler does not bin-pack against PID headroom. Monitor node/process pressure and preserve capacity separately.

## Production rollout checklist

1. Build namespace budgets from historical usage, SLOs, bursts, replica limits, storage growth, and failure scenarios.
2. Test representative Deployments, Jobs, CronJobs, sidecars, init containers, and PVCs in an audit or test namespace.
3. Use server-side dry run and policy tests to verify the defaulted object, min/max, ratios, quota scopes, and expected rejection cases.
4. Preserve headroom for rollouts, HPA scale-out, Job retries, surge Pods, incident debugging, and failover; quota should not equal normal usage.
5. Monitor quota `used/hard`, admission rejections, pending/failed Pods, CPU throttling, OOMs, PVC growth, Pod counts, and node `pid.available`.
6. Protect policy objects with RBAC/admission; use review, a canary namespace, rollback, and post-change validation.
7. Policy changes do not retroactively alter existing objects. Define a restart/recreation plan and compare effective specs before and after rollout.

## Troubleshooting by enforcement layer

1. Describe the workload, ReplicaSet, Pod, and PVC, and inspect events for `exceeded quota`, min/max, ratio, or missing request/limit messages.
2. Run `kubectl get resourcequota,limitrange -n <namespace> -o yaml`; compare quota `status.used`/`status.hard` and LimitRange defaults/constraints.
3. If a Deployment exists but lacks replicas, inspect ReplicaSet events. Admission of the top-level object does not imply admission of child Pods.
4. If a Pod was admitted but remains `Pending`, inspect requests, node allocatable, taints/topology, and PVC state. Passing quota does not mean capacity is available.
5. If process creation reports `resource temporarily unavailable`, the node is unstable, or `PIDPressure` appears, inspect kubelet configuration, node PID use, per-Pod process counts, and eviction events.
6. Confirm ownership before changing policy. Do not merely raise quota to hide a resource leak, runaway CronJob, or fork bomb.

## Common interview questions and precise answers

### How do LimitRange and ResourceQuota differ?

LimitRange constrains and can default the resource declaration of one object; ResourceQuota caps aggregate namespace consumption and object count. LimitRange commonly fills or validates requests/limits first, after which ResourceQuota checks whether adding the object would exceed the total budget.

### Does ResourceQuota guarantee those resources to a namespace?

No. Quota is a consumption ceiling, not a capacity reservation. A namespace with quota headroom can still fail scheduling because of cluster capacity, topology, taints, storage, attachment limits, or priority.

### Why can a Deployment exist without any Pods?

Admission runs per object. The Deployment can be stored first, and LimitRange/ResourceQuota is triggered later when the ReplicaSet creates a Pod. The rejection reason appears in ReplicaSet or Deployment events.

### Can `pid.available` eviction stop a fork bomb?

Not by itself. Eviction reacts to a periodically sampled signal, so PIDs can be exhausted before the next check. Establish a hard boundary with per-Pod `podPidsLimit`, then add reservations, eviction, monitoring, and runtime hardening.

### Why avoid multiple overlapping LimitRanges?

If multiple LimitRanges in one namespace provide defaults, the selected default is nondeterministic and admission results can differ from team expectations. Production should use single ownership, avoid overlapping defaulting, and test the final object.

## Version and stale-content assessment

- In the Kubernetes v1.36 pages reviewed, `LimitRange`, `ResourceQuota`, PID reservation, `podPidsLimit`, and `pid.available` are current mechanisms; there is no removed API object that needs to be carried forward.
- The official PID page still documents the kubelet `--pod-max-pids` command-line option. This guide retains the concept but uses version-controlled kubelet configuration as the production default. Check provider and version documentation for managed-service support and rollout behavior.
- Local ephemeral-storage quota is not GA and has moved to [Kubernetes Policy Preview Features](policy-preview-features.md), outside this production baseline.
- Check new or version-sensitive quota scopes/resources against the target cluster version. Do not apply current v1.36 website examples unchanged to older clusters.

## Official references

- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Process ID Limits and Reservations](https://kubernetes.io/docs/concepts/policy/pid-limiting/)
