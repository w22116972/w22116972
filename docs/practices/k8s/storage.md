# Kubernetes 儲存：面試深度指南

Kubernetes storage 的核心不是「把 disk mount 進 container」，而是將 application 的資料需求、storage implementation、scheduling topology 與 lifecycle policy 分離。面試時應先分清楚 Pod-scoped volume、PersistentVolume（PV）、PersistentVolumeClaim（PVC）、StorageClass、CSI driver 與 node-local ephemeral storage 的責任，再討論效能、可用性、備份與故障排除。

## 先掌握控制流程與資料路徑

```text
dynamic persistent storage
PVC -> StorageClass -> external CSI provisioner -> storage backend
 |                                               |
 | bind                                          | volume
 v                                               v
PV -> scheduler topology decision -> node -> CSI attach/mount -> Pod

data protection
PVC -> VolumeSnapshot -> CSI snapshotter/driver -> backend snapshot
PVC <- restore or clone into a new volume

node-local ephemeral storage
container writable layer + logs + disk-backed emptyDir -> kubelet accounting/eviction
```

- Kubernetes API objects 描述 desired state；CSI controllers、CSI node plugins、scheduler 與 kubelet 執行 provision、attach、mount、resize 與 cleanup。
- PVC `Bound` 只代表 claim 已綁定 volume，不代表 Pod 已成功 attach、mount、讀寫或通過 application-level validation。
- Storage availability、durability 與 replication 主要由 storage backend/CSI driver 提供；Kubernetes 不會自動將單一 disk 複寫成 high availability storage。

## 儲存抽象與責任邊界

| 物件或機制 | Scope | 主要責任 | 常見誤解 |
| --- | --- | --- | --- |
| Pod volume | Pod | 讓同一 Pod 的 containers 共享或使用資料 | volume 不一定 persistent |
| PV | Cluster | 代表一個可供 claim 使用的 storage resource | PV 不是 namespaced，也不等於實體 disk 本身 |
| PVC | Namespace | 使用者對 capacity、access mode、class、volume mode 的請求 | request 不保證 backend 能供應 |
| StorageClass | Cluster | 定義 provisioner、parameters、binding、reclaim 與 expansion policy | 它描述新 volume 的 provisioning class，不是即時 QoS switch |
| VolumeAttributesClass | Cluster | 由 CSI driver 支援時，描述既有 volume 可變更的 attributes/QoS | 不取代 StorageClass；driver 必須支援 modify-volume |
| CSI driver | Cluster/node extension | 將 Kubernetes lifecycle calls 轉為 vendor storage operations | API object 存在不代表 driver/sidecars healthy |
| VolumeSnapshotClass | Cluster | 指定 snapshot driver、parameters 與 deletion policy | snapshot 不等於獨立、跨故障域的 backup |

## Volume 類型怎麼選

| 需求 | 優先考慮 | Lifecycle / 注意事項 |
| --- | --- | --- |
| Pod 內暫存、cache、跨 container 共享 | `emptyDir` | 隨 Pod 消失；disk-backed 計入 ephemeral storage，`medium: Memory` 計入 memory |
| 注入設定、Secret、identity | `configMap`、`secret`、`projected` | 通常 read-only；更新不是立即且 `subPath` mount 不會接收更新 |
| 長期 application data | PVC + CSI | lifecycle 與 Pod 分離；需設計 topology、reclaim、backup、restore |
| 每個 Pod 需要短生命週期但可動態 provision 的 disk | generic ephemeral volume | inline PVC template；通常隨 Pod 建立與刪除 |
| 綁定特定 node 的本機 disk | local PV | 必須有 `nodeAffinity`；node failure 時資料不可自動搬移 |
| 直接存取 host filesystem | 儘量避免 `hostPath` | 高安全風險、不可攜、與 node lifecycle 綁定 |

對新 storage integration 應使用 CSI，並確認目標 Kubernetes version、CSI driver 與 sidecars 的 compatibility matrix。

## PV、PVC lifecycle 與 access modes

典型 lifecycle 是 `Provisioning -> Binding -> Using -> Reclaiming`：

1. Administrator 預先建立 PV，或 PVC 透過 StorageClass dynamic provisioning 建立 PV。
2. Control plane 依 `storageClassName`、capacity、access modes、`volumeMode` 與 selector 將 PVC/PV 一對一綁定。
3. Pod 引用 PVC；scheduler 與 CSI 確認 topology，kubelet/CSI 執行 attach 與 mount。
4. PVC 刪除後，PV 依 reclaim policy `Delete` 或 `Retain` 處理；finalizer 讓 backend cleanup 完成前不先移除 API object。

Access mode 要精準回答：

- `ReadWriteOnce`（RWO）是 single-node read-write，同一 node 上仍可能有多個 Pods 使用；它不是 single-Pod guarantee。
- `ReadWriteOncePod`（RWOP）才限制整個 cluster 只有一個 Pod read-write mount，且只支援 CSI；Kubernetes 1.29 起 stable。
- `ReadOnlyMany`（ROX）與 `ReadWriteMany`（RWX）是否可用取決於 driver/backend。
- Access modes 主要用於 binding 與 mount constraints，不是 filesystem authorization。即使宣告 ROX，也應以 read-only mount、permissions 與 application controls 驗證實際保護。
- `volumeMode: Filesystem` 由 filesystem mount 使用；`Block` 將 raw block device 交給 application，後者需自行處理 filesystem/data integrity。

## StorageClass、dynamic provisioning 與 topology

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-general
provisioner: csi.example.com
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  encrypted: "true"
```

- `provisioner` 與 `parameters` 是 driver-specific contract；升級前要確認 CSI/Kubernetes compatibility matrix。
- `reclaimPolicy` 未指定時預設 `Delete`。重要資料宜明確選擇並測試 `Retain`/`Delete` workflow；`Retain` 仍需要人工或自動化 scrub/reuse 流程。
- `allowVolumeExpansion: true` 只允許 request resize；controller expansion、node expansion 與 filesystem growth 是否 online 完成仍取決於 driver/filesystem。
- `Immediate` 可能在 Pod scheduling 前於錯誤 zone 建立 volume。Topology-constrained storage 通常使用 `WaitForFirstConsumer`，讓 scheduler 綜合 Pod constraints 後再 provision/bind。
- 使用 `WaitForFirstConsumer` 時不要在 Pod 直接設定 `nodeName`，因為它繞過 scheduler，可能使 PVC 一直 `Pending`；使用 node selector/affinity。
- Cluster 最好只有一個 default StorageClass。多個 defaults 時，未指定 class 的 PVC 會使用最近建立的 default，容易造成意外 provisioning。
- `storageClassName: ""` 明確表示不使用 default class；省略欄位可能由 default StorageClass 補上，兩者不同。

## VolumeAttributesClass：變更既有 volume attributes

`VolumeAttributesClass` 在 Kubernetes 1.36 成為 stable。它允許 PVC 透過 CSI driver 將既有 volume 切換到另一組 mutable attributes，例如 storage tier、IOPS 或 throughput，而不必建立新的 StorageClass/PV。

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: high-throughput
driverName: csi.example.com
parameters:
  tier: premium
```

使用前必須確認 API version、feature availability、CSI driver 的 modify-volume support、允許的 transitions、費用與 rollback。把 class name 改到 PVC 是 asynchronous reconciliation，不應在 status 成功前宣告效能已變更；unsupported parameters 或 backend failure 應由 PVC conditions/events 追查。

## Projected volumes 與短期 identity

`projected` volume 可將 stable 的 `Secret`、`ConfigMap`、Downward API 與 `serviceAccountToken` sources 映射到同一 directory。Beta 的 ClusterTrustBundle 與 Pod certificate projections 另見 [Kubernetes Storage Preview Features](storage-preview-features.md)。

```yaml
volumes:
  - name: workload-identity
    projected:
      defaultMode: 0400
      sources:
        - serviceAccountToken:
            path: token
            audience: api.example.com
            expirationSeconds: 3600
        - configMap:
            name: trust-bundle
            items:
              - key: ca.crt
                path: ca.crt
```

- 所有 referenced sources 必須與 Pod 同 namespace；path 不可重疊。
- 優先使用 audience-bound、短效且自動輪替的 projected ServiceAccount token，而不是 long-lived token Secret。
- Kubelet 會更新多數 projected content，但 application 必須重新讀檔；以 `subPath` mount 的 projected/Secret/ConfigMap 不會收到後續更新。
- File projection 不會自動讓 Secret 安全：仍需 least-privilege RBAC、encryption at rest、限制 Pod creation/exec/debug 權限，並避免 log 或 copy secrets。

## Ephemeral volumes 與 local ephemeral storage

兩個名稱相似但不同：

- **Ephemeral volumes** 是 volume lifecycle 類別，包含 `emptyDir`、ConfigMap/Secret/projected、CSI ephemeral 與 generic ephemeral volumes。
- **Local ephemeral storage** 是 node resource，主要計算 container writable layers、node-level container logs 與 disk-backed `emptyDir`；不足會造成 eviction。

```yaml
resources:
  requests:
    ephemeral-storage: 1Gi
  limits:
    ephemeral-storage: 2Gi
volumes:
  - name: scratch
    emptyDir:
      sizeLimit: 1Gi
```

- Scheduler 依 requests 放置 Pod；kubelet 依 limits 與 node pressure 管理/evict，但只在受支援的 filesystem layout 下才能正確量測。
- `emptyDir.sizeLimit` 不取代 container `ephemeral-storage` requests/limits；Pod 總量也包含 logs 與 writable layers。
- `emptyDir.medium: Memory` 使用 tmpfs，計入 container memory，而不是 local ephemeral-storage。
- 使用 `Gi`/`Mi` 等正確單位；`400m` 是 0.4 bytes，不是 400 MB。
- Generic ephemeral volumes 由 inline `volumeClaimTemplate` 建立 namespaced PVC；scheduler、quota、StorageClass 與 CSI capacity rules 仍適用。注意 deterministic PVC naming collision 與有 PVC create 權限的使用者可能取得 ownership 的風險。

## Snapshot、restore、clone 與 populator

| 機制 | Source | Result | 關鍵限制 |
| --- | --- | --- | --- |
| VolumeSnapshot | PVC | Backend point-in-time snapshot object/content | 需要 snapshot CRDs、snapshot controller、CSI snapshotter/driver support |
| Restore | VolumeSnapshot | 新 PVC/PV | 新 volume size 不可小於 source；支援度取決於 driver |
| CSI clone | PVC | 獨立的新 PVC/PV | Dynamic CSI provisioner；source/destination 同 namespace、相同 `volumeMode` |
| Custom volume populator | Custom resource | 預先填入資料的新 PVC/PV | v1.33 GA；仍需安裝、操作並監控外部 populator controller |

`VolumeSnapshotClass.deletionPolicy` 決定刪除 `VolumeSnapshot` 時 backend snapshot content 是 `Delete` 或 `Retain`，概念上類似 PV reclaim policy。Snapshot通常與 primary storage 位於相同 failure/administrative domain，因此不是完整 backup；production protection 還要驗證 application consistency、encryption、cross-account/region copy、retention、restore time 與定期 restore drill。

Custom volume populators 已於 v1.33 GA，但 Kubernetes 只提供 data-source contract；實際 population 仍由外部 controller 負責，因此要驗證 ownership、idempotency、failure cleanup、quota、source deletion、restore 與 upgrade compatibility。Cross-namespace data sources 仍為 Alpha，見 [Kubernetes Storage Preview Features](storage-preview-features.md)。

## Capacity、attach limits 與 scheduling

Capacity 有三種不同 failure domain：

1. **Backend capacity**：storage pool/zone 是否能建立所需 volume。CSI driver 可發布 `CSIStorageCapacity`；搭配 `WaitForFirstConsumer` 且 `CSIDriver.spec.storageCapacity: true` 時，scheduler 才會用它篩選 nodes。這只是 size/topology 的粗略檢查，不保證 provisioning 最終成功。
2. **Node attach limit**：cloud/block driver 對單一 node 可 attach 的 volumes 有上限。Scheduler 透過 `CSINode`/driver information 計算；若 driver/node 回報錯誤或 limit 已滿，Pod 可能 `Pending` 或 attach failure。
3. **Node local space**：image layers、logs、writable layers 與 `emptyDir` 消耗 nodefs/imagefs/containerfs，透過 `ephemeral-storage` requests、limits、image garbage collection 與 eviction 管理。

Capacity planning 應同時監控 PVC requested/actual bytes、filesystem usage/inodes、backend quotas/IOPS、snapshot growth、attach count、node ephemeral usage 與 provisioning latency。PVC capacity 尚有空間不代表 IOPS、throughput 或 inode 仍足夠。

## Production design checklist

- 以 workload SLO 決定 block/file/object、latency、IOPS、throughput、access mode、replication 與 failure domain，不只看容量與價格。
- 使用 CSI 並驗證 controller/node components、sidecar versions、RBAC、topology labels 與 upgrade compatibility。
- 明確設定 StorageClass `reclaimPolicy`、`volumeBindingMode`、encryption、expansion 與允許的 topology；不要依賴 provider defaults。
- Stateful workload 跨 zones 時，讓 compute scheduling、volume topology、replication 與 failover procedure 對齊；RWO zonal disk 不會因 Pod 被重建就跨 zone 移動。
- 對 local ephemeral storage 設 requests/limits，控制 log rotation，監控 DiskPressure 與 eviction；不要把重要資料放在 `emptyDir` 或 writable layer。
- Snapshot/backup 要有 retention、immutability、off-domain copy、encryption/key recovery 與 restore validation。只有「snapshot succeeded」不足以證明可恢復。
- 使用 `hostPath`、local PV、raw block、mount propagation 或 privileged storage agents 時，記錄 security boundary 與 admission controls。

## Troubleshooting：沿 lifecycle 定位

```text
PVC request -> class/provisioner -> PV bind -> scheduling/topology
-> attach -> stage/publish mount -> filesystem/application -> resize/snapshot/delete
```

1. **PVC**：查看 phase、conditions、requested class/access mode/size/volume mode，並讀 events；`Pending` 常是無 class、provisioner failure、topology 或 capacity。
2. **StorageClass/driver**：確認 provisioner 名稱、default、binding mode、parameters、CSI controller/node Pods、`CSIDriver`、`CSINode` 與 logs。
3. **Scheduling**：查看 Pod events、node affinity、zone labels、attach limits、`CSIStorageCapacity`；`WaitForFirstConsumer` 不要搭配 `nodeName`。
4. **Attach/mount**：分辨 `FailedAttachVolume`、`FailedMount`、multi-attach、missing device、filesystem、permissions 與 node plugin registration。
5. **Application**：進 Pod 確認 mount point、read/write、UID/GID、`fsGroup`、free bytes/inodes 與 latency；mount 成功不代表 database consistency。
6. **Resize**：確認 PVC requested/status capacity、conditions、StorageClass expansion、controller/node expansion 與 filesystem support；失敗時不要無限增加 request。
7. **Delete/restore**：確認 reclaim/deletion policy、finalizers、backend asset、snapshot readiness 與實際資料 readback，再允許 traffic。

## 常見面試題的精準回答

### PV 和 PVC 為什麼要分開？

PV 是 cluster-scoped supply，PVC 是 namespaced demand。分離後 application 只描述 capacity/access/class，administrator 與 provisioner 管理 backend、policy 和 lifecycle，Pod 也不必直接綁 vendor volume ID。

### `ReadWriteOnce` 是否代表只能有一個 Pod？

不是。RWO 代表同一時間由一個 node read-write mount，同 node 上可能有多個 Pods 使用。需要 cluster-wide single-Pod constraint 時使用 CSI 支援的 `ReadWriteOncePod`，並仍用 application locking 保護資料語意。

### 為什麼使用 `WaitForFirstConsumer`？

它延後 provision/bind 到 scheduler 看到 consuming Pod，讓 volume zone/topology 與 Pod constraints 對齊，避免先在錯誤 zone 建 disk 後 Pod 無法排程。代價是 PVC 在有 consumer 前保持 `Pending`，provisioning latency 也進入 Pod startup path。

### Snapshot 和 backup 有何差別？

Snapshot 是 storage-level point-in-time copy，通常依賴相同 provider/control plane，可能只有 crash consistency。Backup 應有獨立 failure domain、retention/immutability、application consistency、key recovery 與可驗證 restore workflow；snapshot 可以是 backup pipeline 的一環，但不是完整策略。

### PVC `Bound` 為什麼 Pod 還是起不來？

Binding、scheduling、attach、mount 與 application access 是不同 stages。依序查 topology/affinity、node attach limit、CSI controller/node plugin、VolumeAttachment/events、filesystem 與 permissions，而不是只看 PVC phase。

### `VolumeAttributesClass` 和 `StorageClass` 差在哪裡？

StorageClass 主要決定新 volume 如何 provision 與 lifecycle policy；VolumeAttributesClass 在 driver 支援時描述既有 volume 可修改的 attributes。前者選 provisioning profile，後者執行 post-provision modification，兩者都不保證 backend 已完成變更，必須查 status。

## 版本與 feature status

本文件依 2026-08-21 的 Kubernetes current documentation 整理。實際採用前要以目標 cluster version、enabled APIs/feature gates 與 CSI driver matrix 為準。

- 避免 `hostPath`，除非明確需要 node-level access 並有 admission/security controls。
- `VolumeAttributesClass` 在 Kubernetes 1.36 stable；其他 Kubernetes versions 的 API/feature status 可能不同。
- Alpha/Beta storage features 不屬於本 production baseline，集中於 [Kubernetes Storage Preview Features](storage-preview-features.md)。

## 官方參考資料

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Ephemeral Volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Attributes Classes](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Volume Snapshot Classes](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
- [CSI Volume Cloning](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)
- [Volume Populators and Data Sources](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Kubernetes v1.33: Volume Populators Graduate to GA](https://kubernetes.io/blog/2025/05/08/kubernetes-v1-33-volume-populators-ga/)
- [Storage Capacity](https://kubernetes.io/docs/concepts/storage/storage-capacity/)
- [Node-specific Volume Limits](https://kubernetes.io/docs/concepts/storage/storage-limits/)
- [Local Ephemeral Storage](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/)

---

# Kubernetes Storage: Interview-Depth Guide

Kubernetes storage is not merely about mounting a disk into a container. It separates application data requirements, storage implementation, scheduling topology, and lifecycle policy. In an interview, first distinguish Pod-scoped volumes, PersistentVolumes (PVs), PersistentVolumeClaims (PVCs), StorageClasses, CSI drivers, and node-local ephemeral storage; then discuss performance, availability, backup, and troubleshooting.

## Start with the control flow and data path

```text
dynamic persistent storage
PVC -> StorageClass -> external CSI provisioner -> storage backend
 |                                               |
 | bind                                          | volume
 v                                               v
PV -> scheduler topology decision -> node -> CSI attach/mount -> Pod

data protection
PVC -> VolumeSnapshot -> CSI snapshotter/driver -> backend snapshot
PVC <- restore or clone into a new volume

node-local ephemeral storage
container writable layer + logs + disk-backed emptyDir -> kubelet accounting/eviction
```

- Kubernetes API objects describe desired state; CSI controllers, CSI node plugins, the scheduler, and kubelet perform provisioning, attachment, mounting, resizing, and cleanup.
- A `Bound` PVC only proves that a claim is bound to a volume. It does not prove successful Pod attachment, mounting, read/write access, or application-level validation.
- Storage availability, durability, and replication primarily come from the storage backend and CSI driver. Kubernetes does not automatically turn a single disk into highly available storage.

## Storage abstractions and ownership boundaries

| Object or mechanism | Scope | Primary responsibility | Common misconception |
| --- | --- | --- | --- |
| Pod volume | Pod | Share or provide data to containers in one Pod | A volume is not necessarily persistent |
| PV | Cluster | Represent a storage resource available for claims | A PV is not namespaced and is not the physical disk itself |
| PVC | Namespace | Request capacity, access mode, class, and volume mode | A request does not guarantee backend supply |
| StorageClass | Cluster | Define provisioner, parameters, binding, reclaim, and expansion policy | It is a provisioning class, not a live QoS switch |
| VolumeAttributesClass | Cluster | Describe mutable volume attributes/QoS when supported by a CSI driver | It does not replace StorageClass; the driver must support volume modification |
| CSI driver | Cluster/node extension | Translate Kubernetes lifecycle calls into vendor storage operations | API objects do not prove driver or sidecar health |
| VolumeSnapshotClass | Cluster | Select snapshot driver, parameters, and deletion policy | A snapshot is not an independent cross-failure-domain backup |

## Choosing a volume type

| Requirement | Prefer | Lifecycle and caveats |
| --- | --- | --- |
| Pod scratch space, cache, or container sharing | `emptyDir` | Deleted with the Pod; disk-backed data counts as ephemeral storage, while `medium: Memory` counts as memory |
| Configuration, Secrets, or identity | `configMap`, `secret`, `projected` | Usually read-only; updates are delayed and `subPath` mounts do not receive updates |
| Durable application data | PVC plus CSI | Independent of Pod lifecycle; design topology, reclaim, backup, and restore |
| Short-lived dynamically provisioned disk per Pod | Generic ephemeral volume | Inline PVC template, normally created and deleted with the Pod |
| Local disk tied to a node | Local PV | Requires `nodeAffinity`; data does not automatically move after node failure |
| Direct host filesystem access | Avoid `hostPath` where possible | High security risk, non-portable, and tied to node lifecycle |

Use CSI for new storage integrations, and verify the compatibility matrix for the target Kubernetes version, CSI driver, and sidecars.

## PV/PVC lifecycle and access modes

A typical lifecycle is `Provisioning -> Binding -> Using -> Reclaiming`:

1. An administrator pre-creates a PV, or a PVC triggers dynamic provisioning through a StorageClass.
2. The control plane binds one PVC to one PV based on `storageClassName`, capacity, access modes, `volumeMode`, and any selector.
3. A Pod references the PVC; the scheduler and CSI resolve topology, and kubelet/CSI attach and mount it.
4. After PVC deletion, the PV follows its `Delete` or `Retain` reclaim policy. Finalizers keep the API object until backend cleanup completes.

Be precise about access modes:

- `ReadWriteOnce` (RWO) means read-write from one node; multiple Pods on that node can still use it. It is not a single-Pod guarantee.
- `ReadWriteOncePod` (RWOP) constrains read-write mounting to one Pod cluster-wide, is CSI-only, and has been stable since Kubernetes 1.29.
- `ReadOnlyMany` (ROX) and `ReadWriteMany` (RWX) depend on driver and backend support.
- Access modes primarily participate in binding and mount constraints; they are not filesystem authorization. Validate read-only mounts, permissions, and application controls.
- `volumeMode: Filesystem` provides a filesystem mount. `Block` exposes a raw block device, so the application owns filesystem and data-integrity concerns.

## StorageClass, dynamic provisioning, and topology

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-general
provisioner: csi.example.com
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  encrypted: "true"
```

- `provisioner` and `parameters` form a driver-specific contract. Check the CSI/Kubernetes compatibility matrix before upgrades.
- The default `reclaimPolicy` is `Delete`. Select and test `Retain` or `Delete` explicitly for important data; retained storage still needs a scrub and reuse workflow.
- `allowVolumeExpansion: true` permits a resize request. Controller expansion, node expansion, and filesystem growth still depend on the driver and filesystem.
- `Immediate` can create a volume in the wrong zone before Pod scheduling. Topology-constrained storage generally benefits from `WaitForFirstConsumer`, allowing the scheduler to account for Pod constraints before provisioning or binding.
- Do not combine `WaitForFirstConsumer` with Pod `nodeName`, which bypasses the scheduler and can leave the PVC `Pending`; use node selector or affinity.
- Prefer exactly one default StorageClass. If several are defaults, a PVC without a class uses the most recently created default, enabling surprising provisioning.
- `storageClassName: ""` explicitly requests no default class; omitting the field can allow defaulting. These are not equivalent.

## VolumeAttributesClass: modifying an existing volume

`VolumeAttributesClass` became stable in Kubernetes 1.36. When supported by the CSI driver, a PVC can select a set of mutable attributes—such as storage tier, IOPS, or throughput—for an existing volume without creating a new StorageClass and PV.

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: high-throughput
driverName: csi.example.com
parameters:
  tier: premium
```

Before use, verify the API version, feature availability, CSI modify-volume support, valid transitions, cost, and rollback. Changing the class reference on a PVC starts asynchronous reconciliation; do not claim the performance change is complete before status confirms it. Investigate unsupported parameters and backend failures through PVC conditions and events.

## Projected volumes and short-lived identity

A `projected` volume combines the stable `Secret`, `ConfigMap`, Downward API, and `serviceAccountToken` sources into one directory. Beta ClusterTrustBundle and Pod certificate projections are covered in [Kubernetes Storage Preview Features](storage-preview-features.md).

```yaml
volumes:
  - name: workload-identity
    projected:
      defaultMode: 0400
      sources:
        - serviceAccountToken:
            path: token
            audience: api.example.com
            expirationSeconds: 3600
        - configMap:
            name: trust-bundle
            items:
              - key: ca.crt
                path: ca.crt
```

- Referenced sources must be in the Pod's namespace, and paths must not overlap.
- Prefer audience-bound, short-lived, rotating projected ServiceAccount tokens over long-lived token Secrets.
- Kubelet updates most projected content, but applications must reread files. Projected, Secret, and ConfigMap content mounted through `subPath` does not receive later updates.
- File projection does not make a Secret inherently safe. Apply least-privilege RBAC, encryption at rest, restrictions on Pod creation/exec/debug, and prevent secrets from reaching logs or copies.

## Ephemeral volumes and local ephemeral storage

The similar names refer to different concepts:

- **Ephemeral volumes** are a lifecycle category including `emptyDir`, ConfigMap/Secret/projected, CSI ephemeral, and generic ephemeral volumes.
- **Local ephemeral storage** is a node resource mainly consumed by container writable layers, node-level container logs, and disk-backed `emptyDir`; exhaustion can lead to eviction.

```yaml
resources:
  requests:
    ephemeral-storage: 1Gi
  limits:
    ephemeral-storage: 2Gi
volumes:
  - name: scratch
    emptyDir:
      sizeLimit: 1Gi
```

- The scheduler places Pods using requests; kubelet manages limits and node pressure, but it can account correctly only with a supported filesystem layout.
- `emptyDir.sizeLimit` does not replace container `ephemeral-storage` requests and limits. Total Pod use also includes logs and writable layers.
- `emptyDir.medium: Memory` is tmpfs and counts against memory, not local ephemeral storage.
- Use correct units such as `Gi` and `Mi`; `400m` means 0.4 bytes, not 400 MB.
- Generic ephemeral volumes create a namespaced PVC from an inline `volumeClaimTemplate`; scheduler, quota, StorageClass, and CSI capacity rules still apply. Account for deterministic PVC name collisions and the risk that users with PVC create permission can take ownership.

## Snapshot, restore, clone, and populator

| Mechanism | Source | Result | Key constraint |
| --- | --- | --- | --- |
| VolumeSnapshot | PVC | Backend point-in-time snapshot object/content | Requires snapshot CRDs, snapshot controller, and CSI snapshotter/driver support |
| Restore | VolumeSnapshot | New PVC/PV | New volume cannot be smaller than the source; support is driver-specific |
| CSI clone | PVC | Independent new PVC/PV | Dynamic CSI provisioner; same namespace and same `volumeMode` |
| Custom volume populator | Custom resource | New PVC/PV pre-populated with data | GA in v1.33; still requires an external populator controller that is operated and monitored |

`VolumeSnapshotClass.deletionPolicy` controls whether backend snapshot content is deleted or retained when the `VolumeSnapshot` is deleted, similar to PV reclaim policy. Snapshots commonly share the primary storage's failure and administrative domain, so they are not complete backups. Production protection also needs application consistency, encryption, cross-account or cross-region copies, retention, restore-time objectives, and regular restore drills.

Custom volume populators reached GA in v1.33, but Kubernetes provides only the data-source contract. An external controller still performs population, so validate ownership, idempotency, failure cleanup, quota, source deletion, restore, and upgrade compatibility. Cross-namespace data sources remain Alpha and are covered in [Kubernetes Storage Preview Features](storage-preview-features.md).

## Capacity, attachment limits, and scheduling

Capacity has three distinct failure domains:

1. **Backend capacity**: whether a storage pool or zone can create the volume. A CSI driver can publish `CSIStorageCapacity`; the scheduler consumes it only with `WaitForFirstConsumer` and `CSIDriver.spec.storageCapacity: true`. This is a coarse size/topology check, not a provisioning guarantee.
2. **Node attachment limit**: cloud/block drivers limit how many volumes a node can attach. The scheduler uses `CSINode` and driver information; stale or exhausted limits can leave Pods pending or cause attachment failures.
3. **Node-local space**: image layers, logs, writable layers, and `emptyDir` consume nodefs/imagefs/containerfs and are managed with `ephemeral-storage` requests, limits, image garbage collection, and eviction.

Capacity planning should monitor PVC requested and actual bytes, filesystem usage and inodes, backend quotas and IOPS, snapshot growth, attachment count, node ephemeral usage, and provisioning latency. Free PVC bytes do not prove that IOPS, throughput, or inodes are sufficient.

## Production design checklist

- Derive block/file/object choice, latency, IOPS, throughput, access mode, replication, and failure domain from the workload SLO—not only capacity and price.
- Use CSI and verify controller/node components, sidecar versions, RBAC, topology labels, and upgrade compatibility.
- Explicitly set StorageClass reclaim, binding, encryption, expansion, and topology policy; do not rely on provider defaults.
- For multi-zone stateful workloads, align compute scheduling, volume topology, replication, and failover procedures. Recreating a Pod does not move a zonal RWO disk across zones.
- Set local ephemeral-storage requests and limits, rotate logs, and monitor DiskPressure and eviction. Never put important data in `emptyDir` or the writable layer.
- Give snapshots and backups retention, immutability, off-domain copies, encryption/key recovery, and restore validation. A successful snapshot alone does not prove recoverability.
- Document the security boundary and admission controls for `hostPath`, local PV, raw block, mount propagation, and privileged storage agents.

## Troubleshooting along the lifecycle

```text
PVC request -> class/provisioner -> PV bind -> scheduling/topology
-> attach -> stage/publish mount -> filesystem/application -> resize/snapshot/delete
```

1. **PVC**: inspect phase, conditions, requested class/access mode/size/volume mode, and events. `Pending` often means no class, provisioner failure, topology, or capacity.
2. **StorageClass/driver**: verify provisioner name, default, binding mode, parameters, CSI controller/node Pods, `CSIDriver`, `CSINode`, and logs.
3. **Scheduling**: inspect Pod events, node affinity, zone labels, attachment limits, and `CSIStorageCapacity`. Do not use `nodeName` with `WaitForFirstConsumer`.
4. **Attach/mount**: distinguish `FailedAttachVolume`, `FailedMount`, multi-attach, missing device, filesystem, permission, and node-plugin-registration failures.
5. **Application**: inside the Pod, verify mount point, read/write, UID/GID, `fsGroup`, free bytes/inodes, and latency. A mounted filesystem does not prove database consistency.
6. **Resize**: check PVC requested/status capacity, conditions, StorageClass expansion, controller/node expansion, and filesystem support. Do not repeatedly increase the request after failure.
7. **Delete/restore**: verify reclaim/deletion policy, finalizers, backend asset, snapshot readiness, and real data readback before allowing traffic.

## Precise answers to common interview questions

### Why separate PV and PVC?

A PV is cluster-scoped supply; a PVC is namespaced demand. This lets applications request capacity, access, and class while administrators and provisioners own backend policy and lifecycle. Pods do not need vendor volume IDs.

### Does `ReadWriteOnce` mean only one Pod?

No. RWO means read-write mounting from one node; multiple Pods on that node can use it. For a cluster-wide single-Pod constraint, use CSI-supported `ReadWriteOncePod` and still use application locking to protect data semantics.

### Why use `WaitForFirstConsumer`?

It delays provisioning or binding until the scheduler sees the consuming Pod, aligning volume zone/topology with Pod constraints and preventing creation in the wrong zone. The trade-off is that the PVC remains `Pending` until a consumer exists and provisioning latency enters the Pod startup path.

### What is the difference between a snapshot and a backup?

A snapshot is a storage-level point-in-time copy, often dependent on the same provider and control plane and possibly only crash-consistent. A backup needs an independent failure domain, retention and immutability, application consistency, key recovery, and a tested restore workflow. A snapshot can be one stage of a backup pipeline, not the whole strategy.

### Why can a Pod fail after its PVC is `Bound`?

Binding, scheduling, attachment, mounting, and application access are separate stages. Check topology and affinity, node attachment limits, CSI controller/node plugins, VolumeAttachment and events, filesystem, and permissions instead of stopping at the PVC phase.

### How do VolumeAttributesClass and StorageClass differ?

StorageClass primarily controls how a new volume is provisioned and its lifecycle policy. VolumeAttributesClass describes post-provision mutable attributes for an existing volume when the driver supports them. Neither proves that a backend operation completed; inspect status.

## Version and feature-status notes

This guide was reviewed against the current Kubernetes documentation on 2026-08-21. Before adoption, verify the target cluster version, enabled APIs and feature gates, and the CSI driver compatibility matrix.

- Avoid `hostPath` unless node-level access is intentional and protected by admission and security controls.
- `VolumeAttributesClass` is stable in Kubernetes 1.36; other Kubernetes versions can have different API and feature status.
- Alpha and beta storage features are outside this production baseline and are collected in [Kubernetes Storage Preview Features](storage-preview-features.md).

## Official references

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Ephemeral Volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Attributes Classes](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Volume Snapshot Classes](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
- [CSI Volume Cloning](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)
- [Volume Populators and Data Sources](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Kubernetes v1.33: Volume Populators Graduate to GA](https://kubernetes.io/blog/2025/05/08/kubernetes-v1-33-volume-populators-ga/)
- [Storage Capacity](https://kubernetes.io/docs/concepts/storage/storage-capacity/)
- [Node-specific Volume Limits](https://kubernetes.io/docs/concepts/storage/storage-limits/)
- [Local Ephemeral Storage](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/)
