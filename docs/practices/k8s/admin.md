# Kubernetes 叢集管理：可靠性、容量與 Control Plane 面試指南

Cluster administration 的核心不是背誦 flags，而是管理幾條彼此相依的控制迴路：node lifecycle 決定 workload 是否安全遷移，autoscaler 依 schedulability 調整容量，admission 控制寫入 API 的內容，API Priority and Fairness（APF）保護 API server，而 upgrade compatibility 與 leader election 降低 control plane 版本切換風險。

## 先掌握管理責任邊界

```text
API clients
  -> authentication / authorization
  -> API Priority and Fairness: classify, queue or reject
  -> admission: built-in policy, ValidatingAdmissionPolicy, webhooks
  -> persisted desired state
  -> scheduler + controllers
  -> node autoscaler provisions or consolidates capacity
  -> kubelet manages Pods, shutdown, swap and node status
```

| 問題 | 主要機制 | 不應混淆 |
| --- | --- | --- |
| Pending Pods 缺少可用 node | Node autoscaler | HPA 增減 Pods；scheduler 只選 node |
| Planned node maintenance | cordon、drain、kubelet graceful shutdown | PDB 不保證 node 永不關機 |
| API overload | APF | ResourceQuota 管 namespace resources，不管 API concurrency |
| API object policy | built-in admission、CEL policy、webhook | RBAC 決定「可否嘗試」，admission 決定 object 是否可接受 |
| GPU 或特殊 device allocation | Dynamic Resource Allocation（DRA） | CPU/memory requests 與傳統 extended resources 的模型不同 |
| Control plane upgrade behavior | version skew policy、emulation/compatibility controls | binary version 不等於 effective behavior |

## Node shutdown 與 maintenance

Planned maintenance 的 production baseline 是 `cordon` 後 `drain`，確認 workloads 已遷移，再停止或重啟 node。PDB 只限制 voluntary eviction，不保證 power loss、node failure 或所有 shutdown path。Kubelet graceful node shutdown 仍是 Beta；設定、平台差異與 adoption gate 見 [Kubernetes Administration Preview Features](admin-preview-features.md)。

Non-graceful shutdown 時，只有在確認 node 已關機、而非仍在重啟或 partitioned 的前提下，才考慮加入 `node.kubernetes.io/out-of-service` taint。它會 force-delete 不容忍此 taint 的 Pods 並加速 volume detach；若舊 node 其實仍在存取 volume，可能造成 split-brain 或資料毀損。恢復後由操作者移除該 taint。

## Swap memory management

Linux node 的預設 `NoSwap` 表示 Kubernetes workloads 不使用 swap。若 kubelet 設為 `failSwapOn: false`，node 上的 system services 仍可使用 swap；只有明確選擇 `LimitedSwap` 才讓 Pods 使用 swap。不要沿用「Kubernetes 永遠禁止任何 swap」這個舊式絕對說法。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
memorySwap:
  swapBehavior: LimitedSwap
```

Production 採用前應驗證：

- OS、kernel、cgroup v2 與 container runtime 是否支援；不同 nodes 使用不同 behavior 會讓效能與 eviction 行為不一致。
- `/metrics/resource`、`/stats/summary` 與 node status 的 swap signals 是否已進監控。
- Memory-backed `emptyDir`、Secret 等 tmpfs 必須避免被 swap；kernel 6.3 原生支援 `noswap`，舊 kernel 需確認 distro backport，並使用 encrypted swap 降低資料外洩風險。
- Swap 會改變 memory pressure、latency 與 OOM/eviction 時序；先以受控 workload 壓測，不把 swap 當成 request/limit sizing 的替代品。

## Node autoscaling

Node autoscaler 的首要 input 是 Pending Pods 的 scheduling constraints 與 autoscaler 的 node constraints。它依 resource **requests**、affinity、taints、topology、volume 與可供應的 node shapes 預測 schedulability，不直接依 Pod 實際 CPU/memory usage 增加 node。

```text
load increases
  -> HPA creates more Pods
  -> Pods become Pending because requests cannot fit
  -> node autoscaler provisions capacity
  -> scheduler binds Pods

underutilized capacity
  -> autoscaler simulates rescheduling
  -> respects disruption and scheduling constraints
  -> drains and consolidates nodes
```

面試重點：

- HPA 與 node autoscaler 是互補 control loops；前者調 replica，後者提供可排程容量。
- Provisioning 可能因 cloud capacity、quota、node-pool limits、unsupported constraints 或 volume topology 失敗。
- Consolidation 也依 requests 而非 real usage。Requests 過高會阻止 consolidation，過低則可能使 node 看似可移除但 runtime 壓力過高。
- Non-empty node consolidation 是 disruptive；PDB、termination grace period、local storage、DaemonSets、topology spread 與 replacement capacity 都會影響結果。
- Autoscaler 只能預測 scheduler 決策，不能保證兩者之間沒有新 Pod 或狀態競爭；監控 Pending duration、provisioning errors、node launch latency 與 disruption rate。

## Admission webhook good practices

優先順序應是 built-in admission、CRD schema/defaulting、ValidatingAdmissionPolicy（CEL），最後才是需要遠端程式邏輯的 webhook。Webhook 位於 API write path，錯誤的 availability、scope 或 latency 設計會成為 cluster-wide outage amplifier。

```yaml
webhooks:
  - name: mutate.example.com
    failurePolicy: Ignore
    timeoutSeconds: 2
    sideEffects: None
    matchPolicy: Equivalent
    namespaceSelector:
      matchExpressions:
        - key: admission.example.com/enabled
          operator: In
          values: ["true"]
```

- Mutating webhooks 串行執行且可能 reinvoke；validating webhooks 可平行。讓 mutation minimal、idempotent，且不要依賴 invocation order。
- 用 `namespaceSelector`、`objectSelector`、rules 與 CEL `matchConditions` 縮小 scope，排除 webhook 自己、kube-system 與它依賴的 CNI/CSI 等 add-ons，避免 bootstrap deadlock。
- 使用小 timeout、multiple replicas、topology spread 與 load balancing，觀測 latency、timeouts、rejections 及 API server audit annotations。
- Mutating webhook 通常可 `failurePolicy: Ignore`，再由 validating policy/controller 驗證 final state；security-critical validating control 是否 fail closed，則需依 threat model 與 availability objective 決定。
- Dry-run 時不得產生 side effects；不要覆寫 arrays 或無關欄位；用 staging、server-side dry run 與 minor-version upgrade tests 驗證所有 webhooks 的組合集合仍 idempotent。
- 漸進式以 test namespace rollout；嚴格限制誰能修改 `MutatingWebhookConfiguration`、`ValidatingWebhookConfiguration` 與 webhook serving namespace。

## Dynamic Resource Allocation（DRA）

DRA 讓 scheduler 依 `DeviceClass`、`ResourceClaim`/`ResourceClaimTemplate` 與 driver 發布的 `ResourceSlice` 選擇 devices。核心 DRA 已 stable；非 GA authorization extensions 另見 [Kubernetes Security Preview Features](security-preview-features.md)。採用前仍須查 API version、driver support 與 upgrade path。

管理 guardrails：

- `DeviceClass` 與 `ResourceSlice` 只授權 admins 與 DRA drivers；tenant operators 只取得其 namespace 中必要的 `ResourceClaim` 與 `ResourceClaimTemplate` 權限。
- 對 `ResourceClaim/status` 更新使用最小權限；stable baseline 不應假設 preview synthetic subresources 一定存在。
- `adminAccess` 能存取使用中的 device，是 privileged maintenance mode；只在明確標記並受限的 namespaces 開放，不授予一般 tenants。
- 驗證 scheduler、kubelet、API server 與 driver 的版本相容性，以及 driver failure、device unhealthy、claim deletion、node drain 和 restore 行為。
- DRA device preemption 仍有限制；高優先級 Pod 可能持續 Pending，不能假設 PriorityClass 一定能搶占 device。

## API Priority and Fairness

APF 在 v1.29 已 stable。它以 `FlowSchema` 分類 requests，再由 `PriorityLevelConfiguration` 分配 concurrency seats、queue 或 reject，避免大量低優先級 API calls 壓垮 control plane。`Queue` 以 fair queuing 與 shuffle sharding 隔離 flows；`Reject` 對超量 requests 回 `429 Too Many Requests`。

- APF 管的是 API request concurrency，不是 workload CPU/memory quota，也不是 client-side rate limiting 的替代品。
- Watch requests 的 seat accounting 與一般短 requests 不同；大量 controllers、LIST/WATCH storms 與慢 webhook 都應納入容量分析。
- 保留 mandatory objects 與極小的 `exempt` scope。把一般 automation 放入 exempt 會繞過保護並可能餓死 control plane。
- 變更前保存現況，以 generated load 或 staging 驗證 `apiserver_flowcontrol_*` metrics、queue wait、rejected requests 與各 priority level 的 fairness。
- 收到 `429` 時 client 應尊重 backoff；先判斷是正常 overload protection、錯誤分類，或 controller request storm。

## Upgrade compatibility 與 leader election

Kubernetes v1.32 起，control plane components 可用 emulation/compatibility controls 將 effective capabilities 對齊較舊版本，讓 binary rollout 與 behavior enablement 分成較小步驟。它是進階 upgrade 工具，不取代官方 version skew policy、etcd/API backups、staging rehearsal 與 rollback plan。

Coordinated leader election 在 v1.36 仍非 GA，因此不屬於本 production baseline；其機制、風險與 adoption gate 見 [Kubernetes Administration Preview Features](admin-preview-features.md)。Stable leader election 仍只協調 active replica，不保證 etcd quorum、API endpoint、network path 或 component state 本身健康。

## Certificates 與 networking 的 scope

本批次提供的 Certificates URL 目前只是導向 task 文件的薄頁，最後實質更新日期為 2021；不應由該頁衍生新的操作做法。本 repo 已由 [Kubernetes PKI and Certificate Best Practices](k8s-pki-certificate-bp.md) 說明 CA trust、rotation、expiry 與 kubeadm/managed-service 邊界。

Cluster Networking 頁面仍正確描述 container-to-container、Pod-to-Pod、Pod-to-Service、external-to-Service 四類問題，但詳細 Service、CNI、NetworkPolicy、EndpointSlice、DNS、dual-stack 與 traffic policy 已集中於 [Kubernetes Networking](networking.md)。管理者在這裡只需記得：Kubernetes 定義 network model，實際 data plane、IPAM 與 policy enforcement 由 CNI/其他實作提供。

## 故障排查順序

1. 先界定是 API admission/flow control、scheduler、autoscaler、kubelet、OS 還是 cloud/device provider 問題。
2. 比對 desired configuration 與 effective runtime state：component flags/config、feature gates、API versions、webhook configurations、events、metrics 與 logs。
3. Pending Pod 先看 scheduler events 與 requests/constraints，再看 autoscaler decisions 與 cloud capacity；不要先怪 autoscaler。
4. API latency 依序看 APF queue/rejection、webhook latency、etcd latency、LIST/WATCH volume 與 client retries。
5. Node maintenance 要確認 workload replacement、PDB、volume detach、node power state 與 taints；不可只看 Node `NotReady`。
6. 變更使用 staged rollout、明確 rollback trigger 與 post-change validation，避免同時調整 admission、APF 與 autoscaling 而失去因果關係。

## 常見面試題

### HPA 和 node autoscaler 有何差異？

HPA 根據 metrics 調整 workload replicas；node autoscaler 根據 Pods 是否能滿足 scheduling constraints 來 provision/consolidate nodes。HPA 可能先產生 Pending Pods，這些 Pods 的 requests 才成為 node autoscaler 的容量訊號。

### Webhook fail-open 是否代表安全性降低？

取決於責任分層。Mutating webhook fail-open 可保護 availability，但 critical invariant 應由 built-in admission、CEL 或高可用 validating layer 驗證 final state。不能只改 `failurePolicy`，而不設 compensating validation、monitoring 與 reconciliation。

### APF 和 `--max-requests-inflight` 有何差異？

簡單的 global in-flight limits 只提供整體上限；APF 會分類 traffic、分配 concurrency、queue/reject 並在 flows 間提供 fairness，避免單一 noisy client 佔滿 API server。

### Graceful shutdown 為什麼不能取代 drain？

Graceful shutdown 是 kubelet 對 OS shutdown signal 的最後一道保護；drain 是操作者可觀測、可驗證的 maintenance workflow，能在關機前處理 eviction、PDB、local data 與 replacement readiness。

## 版本與內容新鮮度

- 本指南於 2026-08-21 依 Kubernetes v1.36 文件檢視；部署時仍須對照實際 cluster minor version 與 provider support。
- Alpha/Beta administration features 已移到 supplementary guides；本文件只保留 stable DRA core 與一般 leader-election responsibility boundary。
- 未採用 legacy 指引：不宣稱 Kubernetes 永遠禁止 swap、不把 deprecated scale-up/scale-down 名詞當成唯一術語，也不從 2021 Certificates 薄頁複製設定。
- Feature state、API group 與 flags 會隨 minor release 改變；所有非-stable 功能在 upgrade 前都要重新查證。

## 參考資料

- [Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Swap memory management](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [Certificates](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
- [Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Compatibility Version For Kubernetes Control Plane Components](https://kubernetes.io/docs/concepts/cluster-administration/compatibility-version/)
- [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
- [Coordinated Leader Election](https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)

---

# Kubernetes Cluster Administration: Reliability, Capacity, and Control Plane Interview Guide

Cluster administration is not about memorizing flags. It is about managing interdependent control loops: node lifecycle determines whether workloads move safely, autoscalers adjust capacity from schedulability signals, admission controls API writes, API Priority and Fairness (APF) protects the API server, and upgrade compatibility plus leader election reduce control-plane transition risk.

## Start with the responsibility boundaries

```text
API clients
  -> authentication / authorization
  -> API Priority and Fairness: classify, queue or reject
  -> admission: built-in policy, ValidatingAdmissionPolicy, webhooks
  -> persisted desired state
  -> scheduler + controllers
  -> node autoscaler provisions or consolidates capacity
  -> kubelet manages Pods, shutdown, swap and node status
```

| Problem | Primary mechanism | Do not confuse it with |
| --- | --- | --- |
| Pending Pods lack a suitable node | Node autoscaler | HPA changes Pods; the scheduler only selects nodes |
| Planned node maintenance | cordon, drain, kubelet graceful shutdown | A PDB does not guarantee that a node never shuts down |
| API overload | APF | ResourceQuota governs namespace resources, not API concurrency |
| API object policy | built-in admission, CEL policy, webhook | RBAC permits an attempt; admission decides whether the object is acceptable |
| GPU or special-device allocation | Dynamic Resource Allocation (DRA) | CPU/memory requests and legacy extended resources have different models |
| Control-plane upgrade behavior | version skew policy and emulation/compatibility controls | Binary version is not necessarily effective behavior |

## Node shutdown and maintenance

The production baseline for planned maintenance remains `cordon`, then `drain`, verify workload replacement, and only then stop or restart the node. PDBs constrain voluntary eviction; they do not guarantee protection from power loss, node failure, or every shutdown path. Kubelet graceful node shutdown remains Beta; configuration, platform differences, and adoption gates are covered in [Kubernetes Administration Preview Features](admin-preview-features.md).

For non-graceful shutdown, consider the `node.kubernetes.io/out-of-service` taint only after confirming that the node is powered off, not rebooting or merely partitioned. It force-deletes Pods that do not tolerate the taint and accelerates volume detach. If the old node is still accessing the volume, this can cause split-brain or data corruption. The operator must remove the taint after recovery.

## Swap memory management

The default Linux-node behavior, `NoSwap`, prevents Kubernetes workloads from using swap. With `failSwapOn: false`, host system services can still use swap; only explicit `LimitedSwap` lets Pods use it. The old absolute statement that “Kubernetes always forbids all swap” is no longer accurate.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
memorySwap:
  swapBehavior: LimitedSwap
```

Before production adoption, verify:

- OS, kernel, cgroup v2, and runtime support. Mixed node behaviors create inconsistent performance and eviction semantics.
- Swap signals from `/metrics/resource`, `/stats/summary`, and node status are monitored.
- Memory-backed `emptyDir` and Secret tmpfs data cannot be swapped. Kernel 6.3 natively supports `noswap`; older kernels require distro-backport verification, and encrypted swap reduces disclosure risk.
- Swap changes memory pressure, latency, and OOM/eviction timing. Load-test it and never use it as a substitute for correct request and limit sizing.

## Node autoscaling

A node autoscaler's main inputs are Pending Pod scheduling constraints and autoscaler node constraints. It predicts schedulability from resource **requests**, affinity, taints, topology, volumes, and available node shapes; it does not directly provision from observed Pod CPU or memory usage.

```text
load increases
  -> HPA creates more Pods
  -> Pods become Pending because requests cannot fit
  -> node autoscaler provisions capacity
  -> scheduler binds Pods

underutilized capacity
  -> autoscaler simulates rescheduling
  -> respects disruption and scheduling constraints
  -> drains and consolidates nodes
```

Interview points:

- HPA and node autoscaling are complementary loops: one changes replicas, the other supplies schedulable capacity.
- Provisioning can fail because of cloud capacity, quotas, node-pool limits, unsupported constraints, or volume topology.
- Consolidation also uses requests, not actual usage. Oversized requests block it; undersized requests can make removal look safe despite runtime pressure.
- Consolidating non-empty nodes is disruptive. PDBs, termination grace, local storage, DaemonSets, topology spread, and replacement capacity all matter.
- The autoscaler predicts but does not control scheduler decisions. Monitor Pending duration, provisioning errors, launch latency, and disruption rate.

## Admission webhook good practices

Prefer built-in admission, CRD schema/defaulting, and ValidatingAdmissionPolicy (CEL), then use a webhook only for logic that needs a remote program. A webhook sits in the API write path; poor availability, scope, or latency design can amplify into a cluster-wide outage.

```yaml
webhooks:
  - name: mutate.example.com
    failurePolicy: Ignore
    timeoutSeconds: 2
    sideEffects: None
    matchPolicy: Equivalent
    namespaceSelector:
      matchExpressions:
        - key: admission.example.com/enabled
          operator: In
          values: ["true"]
```

- Mutating webhooks run serially and may be reinvoked; validating webhooks can run in parallel. Keep mutation minimal and idempotent, and never depend on invocation order.
- Narrow scope with `namespaceSelector`, `objectSelector`, rules, and CEL `matchConditions`. Exclude the webhook itself, system namespaces, and dependent CNI/CSI add-ons to prevent bootstrap deadlocks.
- Use a small timeout, multiple replicas, topology spread, and load balancing. Observe latency, timeouts, rejections, and API server audit annotations.
- A mutating webhook can often use `failurePolicy: Ignore`, with final state enforced by validation. Whether a security-critical validator fails closed depends on its threat model and availability objective.
- Dry-run must not cause side effects. Do not overwrite arrays or unrelated fields. Test collection-wide idempotence with staging, server-side dry run, and minor-version upgrade tests.
- Roll out by test namespace and tightly restrict modification of webhook configurations and the serving namespace.

## Dynamic Resource Allocation (DRA)

DRA lets the scheduler select devices through `DeviceClass`, `ResourceClaim`/`ResourceClaimTemplate`, and driver-published `ResourceSlice` objects. Core DRA is stable; non-GA authorization extensions are covered in [Kubernetes Security Preview Features](security-preview-features.md). Still verify API versions, driver support, and upgrade paths.

Administrative guardrails:

- Restrict `DeviceClass` and `ResourceSlice` to administrators and drivers. Tenant operators should receive only necessary access to namespaced claims and templates.
- Apply least privilege to `ResourceClaim/status`; the stable baseline must not assume preview synthetic subresources exist.
- `adminAccess` can access an in-use device and is privileged maintenance functionality. Allow it only in explicitly labeled and restricted namespaces.
- Test scheduler, kubelet, API server, and driver compatibility, plus driver failure, unhealthy devices, claim deletion, drain, and restoration.
- DRA device preemption remains limited. A high-priority Pod can remain Pending instead of preempting an allocated device.

## API Priority and Fairness

APF has been stable since v1.29. A `FlowSchema` classifies requests, and a `PriorityLevelConfiguration` allocates concurrency seats, queues, or rejects. `Queue` uses fair queuing and shuffle sharding to isolate flows; `Reject` returns `429 Too Many Requests` for excess work.

- APF manages API concurrency, not workload resource quota, and does not replace client-side rate limiting.
- Watch seat accounting differs from short requests. Controller storms, LIST/WATCH volume, and slow webhooks belong in capacity analysis.
- Preserve mandatory objects and keep the `exempt` scope tiny. Exempting ordinary automation bypasses protection and can starve the control plane.
- Before changes, save current configuration and validate `apiserver_flowcontrol_*` metrics, queue wait, rejection, and fairness under staged load.
- Clients receiving `429` should back off. Determine whether it is healthy overload protection, wrong classification, or a request storm.

## Upgrade compatibility and leader election

Since Kubernetes v1.32, control-plane components can use emulation and compatibility controls to align effective capabilities with an older version, separating binary rollout from behavior enablement. This advanced upgrade tool does not replace the version skew policy, backups, staging rehearsal, or rollback planning.

Coordinated leader election is not GA in v1.36 and is outside this production baseline. Its mechanism, risk, and adoption gate are covered in [Kubernetes Administration Preview Features](admin-preview-features.md). Stable leader election still coordinates only an active replica; it does not guarantee etcd quorum, API endpoint reachability, network health, or component state correctness.

## Certificate and networking scope

The supplied Certificates URL is currently a thin redirect to a task guide and was last materially updated in 2021, so it is not a basis for new operational guidance. This repository already covers CA trust, rotation, expiry, and kubeadm/managed-service boundaries in [Kubernetes PKI and Certificate Best Practices](k8s-pki-certificate-bp.md).

The Cluster Networking page still correctly frames container-to-container, Pod-to-Pod, Pod-to-Service, and external-to-Service problems. Detailed Service, CNI, NetworkPolicy, EndpointSlice, DNS, dual-stack, and traffic policy material belongs in [Kubernetes Networking](networking.md). Kubernetes defines the network model; CNI and other implementations supply the data plane, IPAM, and enforcement.

## Troubleshooting order

1. Locate the failing boundary: API admission/flow control, scheduler, autoscaler, kubelet, OS, or cloud/device provider.
2. Compare desired configuration with effective runtime state: flags/config, gates, APIs, webhook configurations, events, metrics, and logs.
3. For a Pending Pod, inspect scheduler events and constraints before autoscaler decisions and cloud capacity.
4. For API latency, inspect APF queues/rejections, webhook latency, etcd latency, LIST/WATCH volume, and retries.
5. For maintenance, verify replacement workloads, PDBs, volume detach, power state, and taints; `NotReady` alone is insufficient.
6. Stage changes with rollback triggers and post-change validation. Avoid changing admission, APF, and autoscaling simultaneously.

## Common interview questions

### How do HPA and node autoscaling differ?

HPA changes replicas from metrics. Node autoscaling provisions or consolidates nodes according to whether Pods satisfy scheduling constraints. HPA can create Pending Pods whose requests then become the node autoscaler's capacity signal.

### Does webhook fail-open necessarily weaken security?

It depends on layering. Fail-open mutation protects availability, while critical invariants can be enforced by built-in admission, CEL, or a highly available validating layer. Changing only `failurePolicy` without compensating validation, monitoring, and reconciliation is incomplete.

### How is APF different from `--max-requests-inflight`?

A global in-flight limit only caps the total. APF classifies traffic, allocates concurrency, queues or rejects, and provides fairness so one noisy client cannot monopolize the API server.

### Why does graceful shutdown not replace drain?

Graceful shutdown is kubelet's final safeguard for an OS shutdown signal. Drain is an observable maintenance workflow that handles eviction, PDBs, local data, and replacement readiness before power-off.

## Version and freshness notes

- Reviewed on 2026-08-21 against Kubernetes v1.36 documentation; always match the deployed minor version and provider support.
- Alpha and beta administration features have moved to supplementary guides; this document retains only stable DRA core behavior and the general leader-election responsibility boundary.
- Legacy guidance omitted: the guide does not claim Kubernetes always forbids swap, does not treat old scale-up/scale-down names as the only terminology, and does not copy configuration from the 2021 Certificates stub.
- Recheck every non-stable feature state, API group, and flag before an upgrade.

## References

- [Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Swap memory management](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [Certificates](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
- [Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Compatibility Version For Kubernetes Control Plane Components](https://kubernetes.io/docs/concepts/cluster-administration/compatibility-version/)
- [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
- [Coordinated Leader Election](https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)
