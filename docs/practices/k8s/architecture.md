# Kubernetes 架構：面試深度指南

Kubernetes 不是「負責啟動 containers 的單一程式」，而是一個以 API 為中心的分散式控制系統。使用者宣告 desired state，control plane 將狀態儲存在 etcd；scheduler、controllers 與各 node 上的 kubelet 持續協調 current state，使其逐步接近 desired state。

## 先掌握完整控制流程

```text
kubectl / CI / controller
          |
          | HTTPS: authentication -> authorization -> admission
          v
    kube-apiserver <----------> etcd
       |      ^                 cluster state
       |      |
       |      +--- kube-controller-manager
       |      +--- cloud-controller-manager
       |      +--- kube-scheduler
       |
       +------> kubelet ------> container runtime (CRI)
                    |
                    +---------> Pod containers

Service traffic -> Service implementation (kube-proxy or equivalent) -> ready endpoints
Pod networking  -> CNI implementation
Persistent data -> CSI implementation
```

以建立 Deployment 為例：

1. Client 將宣告送到 `kube-apiserver`；請求通過 authentication、authorization、admission 與 validation 後，物件才持久化到 etcd。
2. Deployment controller 觀察到新的 desired state，建立或更新 ReplicaSet；ReplicaSet controller 再建立所需的 Pods。
3. `kube-scheduler` 只處理尚未綁定 node 的 Pod，依 resources、constraints、affinity、topology 與 policy 選擇 node，然後寫回 binding 決策；它不啟動 container。
4. 目標 node 上的 kubelet 觀察到 PodSpec，透過 CRI 要求 container runtime 建立 containers，並回報 Pod 與 Node status。
5. EndpointSlice controller 根據可服務的 Pods 更新 endpoints；Service 的資料平面將流量導向這些 endpoints。

關鍵觀念：元件通常不直接彼此下命令，而是透過 API objects、watch 與 status 進行鬆耦合協調。watch 中斷或事件重複都可能發生，因此 controller 必須能重新列舉狀態、重試並保持 idempotent。

## Control plane 元件與責任邊界

| 元件 | 核心責任 | 面試中應說清楚的限制 |
| --- | --- | --- |
| `kube-apiserver` | Kubernetes API 的入口；執行驗證與 admission，提供讀寫及 watch | 是 control plane 的 hub，可水平擴展；不是所有業務流量的 data plane |
| `etcd` | 儲存 Kubernetes API 的權威狀態 | 一致且高可用不等於不需要備份；etcd 的 latency 與健康會直接影響 API/control loops |
| `kube-scheduler` | 為未綁定的 Pod 選擇 node | 做 placement decision，不負責啟動 Pod，也不負責 node autoscaling |
| `kube-controller-manager` | 執行 Deployment、ReplicaSet、Node、Job、EndpointSlice 等 control loops | reconciliation 通常是 eventual，不是一次交易完成整個流程 |
| `cloud-controller-manager` | 將 Node、Route、Service 等 Kubernetes intent 轉成 cloud API 操作 | 只有 cloud 整合需要；它把 provider-specific logic 與核心 Kubernetes 解耦 |

HA control plane 通常執行多個 API server instances。`kube-scheduler` 與 `kube-controller-manager` 也可執行多個 replicas，但透過 Lease leader election 讓一個 leader 主動工作，其餘待命，避免相同 controller 同時做出衝突操作。是否能承受故障仍取決於 API endpoint、etcd quorum、failure-domain 分布及 component health，而不只是 replica 數量。

## Node 與 workload 執行

- Node 是 API object，也是實際 VM 或實體機器的表示；兩者必須以唯一名稱正確對應。
- kubelet 確保指派給該 node 的 Pod containers 存在且健康，但不管理非 Kubernetes 建立的 containers。
- container runtime 透過 CRI 執行 image pull、container lifecycle 等工作；常見實作包括 containerd 與 CRI-O。
- CNI 負責 Pod networking；CSI 負責 storage integration。這些 extension interfaces 是 Kubernetes 可替換基礎設施的關鍵。
- `kube-proxy` 實作部分 Service 行為，但它是 optional；某些 network plugins 可提供等價的 Service forwarding。

Node health 不是由 scheduler 主動探測。kubelet 會更新 Node status，並更頻繁地更新 `kube-node-lease` namespace 中同名 Lease 的 `spec.renewTime`。control plane 依 heartbeat 判斷 node availability，再由 node lifecycle controller taint 或驅逐 workloads。短暫 heartbeat 延遲不代表立即重排，因為系統需要容忍 transient network failure，並避免不必要的 failover。

## Controller 與 reconciliation

Controller 的基本邏輯是：

```text
observe desired state + current state
              |
              v
       calculate the difference
              |
              v
 perform an idempotent action or update the API
              |
              +---- retry until converged
```

`spec` 通常表達 desired state，`status` 通常回報 observed state。這不是保證每個物件都具有完全相同欄位，而是一個重要 API convention。

可靠 controller 應該：

- 以 level-based state 為主，不假設每一個 event 都會被完整且只處理一次。
- 將操作設計成 idempotent，對 API conflict、timeout、rate limit 與外部系統失敗進行 bounded retry/backoff。
- 只管理自己擁有的資源，使用 labels/selectors 找集合，用 `ownerReferences` 表達 lifecycle ownership。
- 更新 `status`、conditions 與 Events，讓操作人員能區分「尚未收斂」和「無法收斂」。
- 在外部系統建立資源時使用 finalizer，完成 cloud/database 等外部清理後才移除 finalizer。

Operator pattern 只是把同一套 controller/reconciliation 模型延伸到 Custom Resources，並不是另一種架構。

## Control plane 與 node 通訊及信任邊界

Kubernetes 採 hub-and-spoke API pattern：node、Pod 與 control-plane components 對叢集狀態的 API 使用都終止於 `kube-apiserver`，其他 control-plane components 不應暴露成遠端 API hub。

主要路徑：

- **Node/Pod -> API server**：通常使用 HTTPS。kubelet 使用 client credential；Pod 使用有範圍限制的 ServiceAccount identity，並應以 RBAC 遵循 least privilege。
- **API server -> kubelet**：支援 logs、exec/attach 與 port-forward。必須啟用 kubelet authentication/authorization，並驗證 kubelet serving certificate；不要假設反向路徑天生安全。
- **Control plane -> cluster network**：若 control plane 與 nodes 位於分離網路，可使用 Konnectivity 建立由 node-side agents 發起的通道。上游列出的 SSH tunnel 已 deprecated，不應作為新架構建議。

面試時不要把 `kubectl exec` 描述成 kubectl 直接連到 container。請求先到 API server，再經 API server 到 kubelet，最後由 kubelet/runtime 處理 execution stream。

## Cloud controller 的架構價值

`cloud-controller-manager` 把 provider-specific reconciliation 從 Kubernetes 核心元件分離，讓 cloud provider 能獨立演進。典型 controllers 包括：

- Node controller：同步 provider ID、region/zone、addresses，並確認失聯 VM 是否已在 cloud 端刪除。
- Route controller：在需要時配置跨 node Pod network routes。
- Service controller：將 `type: LoadBalancer` 的 Service 調和為 provider load balancer、addresses 與 health-check resources。

這仍是 asynchronous reconciliation。建立 Service 後，cloud load balancer 可能需要時間才可用；應觀察 Service status、Events、controller logs 與 cloud API errors，而不是把 API object 已接受誤認為外部資源已完成。

## Self-healing 的能力與邊界

Kubernetes 的 self-healing 是多個 control loops 的結果：

- kubelet 依 `restartPolicy` 重啟失敗 container。
- workload controller 補足 Deployment、StatefulSet、DaemonSet 等所需 replicas。
- node failure 後，controller 可在其他可用 nodes 建立 replacement Pods。
- readiness 失敗的 Pod 會從 Service endpoints 移除，避免繼續接收一般流量。
- storage controllers 可協調 volume detach/attach，但受 storage topology、provider 狀態與 access mode 限制。

Self-healing 不代表：

- 修正 application bug、corrupt data 或錯誤設定。
- 保證 stateful workload 無資料遺失或立即跨 zone 復原。
- liveness probe 越積極越好；錯誤 probe 可能造成 restart loop 或 cascading failure。
- 單一 Pod 自動成為高可用；需要 replicas、spread、disruption policy、capacity 與應用程式層復原設計。

## Ownership、finalizer 與 garbage collection

Labels/selectors 回答「哪些物件屬於這個集合」；`ownerReferences` 回答「誰控制 dependent 的 lifecycle」。兩者不可互換。

- **Background deletion（預設）**：API server 先刪除 owner，garbage collector 在背景刪除 dependents。
- **Foreground deletion**：owner 保留 `deletionTimestamp` 與 `foregroundDeletion` finalizer，直到會阻擋刪除的 dependents 清理完成。
- **Orphan**：刪除 owner，但刻意保留 dependents。
- **Finalizer**：不是垃圾回收器本身，而是刪除前必須完成的 cleanup contract。finalizer 永遠不移除會讓物件卡在 `Terminating`。

kubelet 也會清理自己管理的 unused containers 與 images；不要部署會與 kubelet lifecycle 競爭的外部清理工具。PersistentVolume 是否隨 PVC 刪除則取決於 reclaim policy，不能只靠 owner reference 推論資料保留策略。

## 故障推理：面試回答框架

遇到「Pod 為什麼沒有恢復」時，沿控制鏈查證，而不是直接重啟：

1. **Intent**：Deployment/StatefulSet 的 spec、replicas、strategy 是否正確？
2. **Admission**：物件是否被 policy/webhook 拒絕或修改？
3. **Controller**：ReplicaSet/Pod 是否建立？conditions、Events、controller 是否有錯誤？
4. **Scheduling**：Pod 是否 Pending？resources、taints、affinity、topology、PVC 是否可滿足？
5. **Node execution**：kubelet、CRI、image pull、CNI、CSI、probes 是否正常？
6. **Serving path**：readiness、EndpointSlice、Service、DNS、ingress/load balancer 是否已收斂？
7. **Dependencies**：Secret、ConfigMap、storage、external API 與 application state 是否可用？

這個順序能定位 failure domain，也能避免把「API 接受物件」、「Pod Running」與「應用程式可服務」混為一談。

## 常見面試題的精準回答

### Scheduler 與 controller 有何差異？

Scheduler 為未綁定的 Pod 選擇 node 並記錄 binding；controller 觀察 desired/current state 的差異並持續建立、更新或刪除資源。scheduler 決定 placement，kubelet 才執行 Pod。

### 如果 scheduler 暫時故障，既有 Pods 會怎樣？

已在 nodes 執行的 Pods 通常繼續由 kubelet 管理；新建立或需要 replacement 的未綁定 Pods 會保持 Pending。這說明 control-plane 元件故障不必然立即停止 data-plane workload，但會影響變更與復原能力。

### API server 或 etcd 故障時會怎樣？

既有 containers 與部分 Service data plane 通常能暫時繼續運作，但新的 API writes、scheduling、controller reconciliation 與最新狀態傳播會受阻。實際影響取決於 failure duration、cached rules、node 狀態與應用程式對 control plane 的依賴。

### Lease 為什麼比只更新 Node status 合適？

Heartbeat 是高頻、輕量的協調更新。獨立 Lease 可降低對大型 Node object 的頻繁寫入，並提供明確的 renew time；Lease 也可用於 HA component leader election。

### Kubernetes 是否是強一致系統？

etcd 對持久化狀態提供一致性基礎，但整個 Kubernetes 行為由多個 asynchronous control loops 組成，端到端收斂是 eventual。面試回答應同時說出這兩層，而不是只選「強一致」或「最終一致」。

## 參考資料

- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
- [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Leases](https://kubernetes.io/docs/concepts/architecture/leases/)
- [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
- [Kubernetes Self-Healing](https://kubernetes.io/docs/concepts/architecture/self-healing/)
- [Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/)

---

# Kubernetes Architecture: Interview-Depth Guide

Kubernetes is an API-centered distributed control system, not one program that starts containers. Users declare desired state, the control plane stores it in etcd, and schedulers, controllers, and node-level kubelets continuously reconcile current state toward that intent.

## End-to-end control flow

Use the architecture diagram and Deployment flow above. The important sequence is:

1. A client sends intent to `kube-apiserver`; authentication, authorization, admission, and validation run before persistence to etcd.
2. The Deployment and ReplicaSet controllers create the required API objects.
3. `kube-scheduler` selects a node for each unbound Pod and records the binding. It does not start containers.
4. The selected node's kubelet observes the PodSpec and asks the runtime through CRI to run the containers.
5. EndpointSlice reconciliation and the Service data plane route traffic only to serving endpoints.

Components generally coordinate through API objects, watches, and status rather than direct commands. Controllers must tolerate interrupted watches and duplicate observations by relisting, retrying, and acting idempotently.

## Responsibilities and boundaries

- `kube-apiserver` is the authenticated and authorized API hub. It scales horizontally but is not the application traffic data plane.
- etcd is the authoritative backing store. Its consistency and availability do not remove the need for tested backups; its latency affects the entire control system.
- `kube-scheduler` makes placement decisions for unbound Pods. It neither starts containers nor performs node autoscaling.
- `kube-controller-manager` runs the built-in reconciliation loops.
- `cloud-controller-manager` separates provider-specific Node, Route, and Service reconciliation from core Kubernetes.

An HA control plane commonly runs multiple API server instances. Scheduler and controller-manager replicas use Lease-based leader election so one leader acts while peers stand by. Availability still depends on the API endpoint, etcd quorum, failure-domain placement, and component health—not replica counts alone.

## Nodes and workload execution

- A Node API object represents a real VM or physical host, and its identity must map uniquely.
- The kubelet maintains Pods assigned to its node and does not manage arbitrary non-Kubernetes containers.
- A CRI runtime such as containerd or CRI-O manages container execution; CNI and CSI integrate networking and storage.
- `kube-proxy` is optional when a network implementation provides equivalent Service forwarding.
- Kubelets update Node status and renew lightweight Leases in `kube-node-lease`. The control plane uses heartbeats to determine availability and then applies lifecycle actions.

A missed heartbeat does not imply immediate rescheduling. Kubernetes must tolerate transient network failure and avoid unsafe or unnecessary failover.

## Reconciliation model

Controllers observe desired and current state, calculate a difference, perform an idempotent action or API update, and retry until the system converges. `spec` commonly expresses intent and `status` reports observation, although exact APIs vary.

Reliable controllers should be level-based, retry conflicts and external failures with backoff, expose conditions, manage only owned resources, and use finalizers when external cleanup must complete before deletion. The Operator pattern extends this same model to Custom Resources.

## Communication and trust boundaries

Kubernetes uses a hub-and-spoke API pattern: node, Pod, and control-plane API traffic terminates at `kube-apiserver`.

- Node and Pod clients normally use HTTPS, scoped identities, and RBAC.
- API server connections to kubelet support logs, exec/attach, and port-forward. Enable kubelet authentication and authorization and validate kubelet serving certificates.
- Konnectivity can bridge separated control-plane and node networks through node-initiated tunnels. Upstream SSH tunnels are deprecated and should not be selected for a new design.

`kubectl exec` does not connect directly to a container: the stream goes through the API server, then kubelet, and finally the runtime.

## Cloud integration

The cloud controller manager decouples provider logic from Kubernetes core. Its Node controller synchronizes provider identity, topology, and addresses; its Route controller may configure Pod network routes; its Service controller reconciles LoadBalancer Services with provider infrastructure.

This is asynchronous reconciliation. An accepted Service object does not prove that the external load balancer is ready; inspect status, Events, controller logs, and provider API errors.

## Self-healing and its limits

Self-healing emerges from multiple loops: kubelet restarts containers according to `restartPolicy`, workload controllers replace replicas, node failure can cause Pods to be recreated elsewhere, readiness removes unhealthy endpoints, and storage controllers coordinate volume movement.

It does not fix application bugs, corrupted data, bad configuration, unavailable storage, or an incorrect probe. A single Pod is not automatically highly available; replicas, topology, disruption policy, spare capacity, and application-level recovery remain design responsibilities.

## Ownership and garbage collection

Labels and selectors define membership; `ownerReferences` define lifecycle ownership.

- Background deletion removes the owner first and dependents asynchronously.
- Foreground deletion retains the terminating owner until blocking dependents are gone.
- Orphan deletion deliberately retains dependents.
- A finalizer is a pre-deletion cleanup contract, not the garbage collector itself. A finalizer that is never removed leaves the object terminating.

The kubelet cleans up containers and images it manages; competing external cleanup can break its lifecycle. PersistentVolume data retention is governed by reclaim policy and must not be inferred only from ownership.

## Interview troubleshooting framework

Trace the control chain: declared intent, admission, controller-created objects, scheduling constraints, kubelet/CRI/CNI/CSI execution, readiness and Service endpoints, then application dependencies. Keep these states distinct: an object was accepted, a Pod is Running, and the application is serving traffic.

## High-value interview answers

- **Scheduler versus controller:** the scheduler selects placement and records a binding; controllers continuously reconcile state; kubelet executes the Pod.
- **Scheduler outage:** existing Pods generally keep running, while new or replacement unbound Pods remain Pending.
- **API server or etcd outage:** existing containers and some cached Service data-plane behavior can continue temporarily, but writes, scheduling, reconciliation, and state propagation are impaired.
- **Why Leases:** they support lightweight, high-frequency heartbeats and HA component leader election without repeatedly updating a large Node object.
- **Is Kubernetes strongly consistent?** etcd provides a consistent persistence foundation, while end-to-end behavior across asynchronous controllers converges eventually. Both statements describe different layers.

## References

- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
- [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Leases](https://kubernetes.io/docs/concepts/architecture/leases/)
- [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
- [Kubernetes Self-Healing](https://kubernetes.io/docs/concepts/architecture/self-healing/)
- [Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/)
