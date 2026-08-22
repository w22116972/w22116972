# Amazon EKS Best Practices：Hybrid Deployments

使用 EKS Hybrid Nodes 時，control plane 在 AWS Region 執行，nodes 則位於 on-premises 或 edge。主要目標是在 wide-area connection 到 control plane 或 regional AWS services 中斷時，仍保持 local applications 穩定。

> 最後檢視：2026-08-20

## 建立 resilient connectivity

- 使用 redundant Direct Connect 或 Site-to-Site VPN paths，並移除 customer gateways、routing、power 與 local networking 的 single points of failure。
  - 兩條連線若共用同一 router、電力或實體線路，仍可能同時故障；redundancy 必須涵蓋完整 end-to-end path。
- 監控 tunnel state、packet loss、latency 與 traffic volume；對 `NodeNotReady` events 發出 alert，判斷故障來自 regional、on-premises 或中間路徑。
- 依 physical site 或 data center 為 hybrid nodes 設定 `topology.kubernetes.io/zone`；cloud controller 不會自動為 hybrid nodes 提供此 label。

## 斷線時保持 applications 穩定

- 在 physical failure domains 間執行多個 replicas 並保留 failover capacity，使用符合實際 site layout 的 topology spread、node selection 與 disruption policy。
- 驗證 disconnection 下的 load balancing 與 ingress；Cilium、Calico、MetalLB、`kube-proxy`、CoreDNS 及 Region-based ALB/NLB paths 行為並不相同。
  - Local data-plane 元件可能繼續轉送既有 traffic，但依賴 AWS Region 的 load balancer、DNS update 或 control-plane reconciliation 可能停止。
- 盤點 ECR、S3、RDS、CloudWatch、Managed Service for Prometheus、regional load balancers、DNS、identity 等 runtime dependencies，必要時提供 cache 或 local alternatives。
  - 判斷依賴是否只在 deploy 時使用，還是每個 request 都需要；後者在 WAN 中斷時會直接影響 application availability。
- 必要時同時送往 remote 與 local observability backends；準備 `crictl` 與 host-level procedures，因 `kubectl` 無法透過 unreachable API server 操作。

## 謹慎調整 pod failover

- 了解 node lifecycle controller 會將 disconnected nodes 標記為 unreachable 並可能 evict pods；正確 zone labeling 很重要。
- 對斷線期間應持續綁定每個 node 的 services 使用 DaemonSets。
- 只有在權衡 local continuity 與 faster failover 後才設定 `node.kubernetes.io/unreachable` 的 `tolerationSeconds`。
  - 時間設太短可能在暫時斷線時誤遷移 pods；設太長則會延後真正 node failure 的 replacement。
- Specialized applications 可由 custom controller 結合 unreachable signal 與 application health，但必須能安全面對 control-plane loss 與 network partitions。
- 測試 full-cluster、full-site、majority-site、minority-site 與 node-restart scenarios，注意 duplicate active instances 與 split-brain。
  - Stateful 或 leader-based applications 必須有 quorum、fencing 或其他 ownership mechanism，避免兩個隔離站點同時接受寫入。

## 管理 host credentials

- 一個 cluster 只使用一種 credential provider：Systems Manager hybrid activations 或 IAM Roles Anywhere，不要兩者並用。
- SSM credentials 短期有效，outage 時 refresh 可能 exponential backoff；保持 agents 更新並記錄 logs、force refresh 的方法。
  - SSM credentials 有效期為一小時；若 refresh 遭遇網路中斷，連線恢復後可能最多約 30 分鐘才重新取得 credentials 並連回 control plane。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/hybrid-nodes-host-creds.html)
- IAM Roles Anywhere 按需取得 credentials，設定 profile duration 對齊 IAM role maximum，並在 certificate expiry 前 alert。
  - IAM Roles Anywhere 預設 credentials 有效期為一小時、最長可設 12 小時，且通常能在網路恢復後數秒內按需重新取得 credentials；代價是必須管理 certificate lifecycle。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/hybrid-nodes-host-creds.html)
- 測試每個 node 的 reconnection timing；staggered recovery 可能讓 pods 從晚恢復的 nodes 移至早恢復的 nodes。

## Review checklist

- Hybrid connectivity 有 redundant paths 及 end-to-end loss/latency alerts。
- Nodes 有 physical zone labels，workloads 橫跨真實 failure domains。
- API server 與 regional dependencies 遺失時，必要 local services 仍可運作。
- 各 application class 的 pod eviction timing 與 split-brain 已測試。
- Host credential refresh 與 post-outage reconnection 有實際演練的 runbook。

---
# Amazon EKS Best Practices: Hybrid Deployments

With EKS Hybrid Nodes, the control plane runs in an AWS Region while nodes run
on-premises or at the edge. The main goal is to keep local applications stable
when the wide-area connection to the control plane or regional AWS services is
interrupted.

## Build resilient connectivity

- Use redundant Direct Connect or Site-to-Site VPN paths and remove single
  points of failure in customer gateways, routing, power, and local networking.
- Monitor tunnel state, packet loss, latency, and traffic volume. Alert on EKS
  controller-manager `NodeNotReady` events and investigate whether the failure
  is regional, on-premises, or on the path between them.
- Assign `topology.kubernetes.io/zone` to hybrid nodes based on physical site or
  data center. Kubernetes does not receive this label automatically from a cloud
  controller for hybrid nodes.

## Keep applications stable while disconnected

- Run multiple replicas across physical failure domains and reserve capacity for
  failover. Use topology spread, node selection, and disruption policy that
  match the actual site layout.
- Validate load balancing and ingress behavior under disconnection. Cilium,
  Calico, MetalLB, `kube-proxy`, CoreDNS, and Region-based ALB or NLB paths do
  not all behave the same when the control plane is unreachable.
- Review every runtime dependency on ECR, S3, RDS, CloudWatch, Managed Service
  for Prometheus, regional load balancers, DNS, identity, and other remote
  services. Cache or provide local alternatives where loss of the dependency
  would break required local operation.
- Configure observability to send to both remote and local backends when needed.
  Prepare `crictl` and host-level procedures because `kubectl` cannot operate
  through an unreachable API server.

## Tune pod failover deliberately

- Understand that the node lifecycle controller marks disconnected nodes
  unreachable and may evict their pods. Kubernetes suppresses some evictions
  when all nodes in a zone are unreachable, which makes correct zone labeling
  important.
- Use DaemonSets for services that should remain bound to every node during a
  disconnection.
- Set `tolerationSeconds` for the `node.kubernetes.io/unreachable` taint only
  after choosing between local continuity and faster failover. A longer value
  keeps a pod bound locally but delays replacement on a reachable node.
- For specialized applications, a custom controller may combine the unreachable
  signal with application health, but it must itself remain safe during control-
  plane loss and network partitions.
- Test full-cluster, full-site, majority-site, minority-site, and node-restart
  scenarios. Watch for duplicate active instances and split-brain behavior.

## Manage host credentials

- Use one credential provider across a cluster: Systems Manager hybrid
  activations or IAM Roles Anywhere, not both.
- SSM credentials are short-lived and refresh can enter exponential backoff
  during an outage. Keep agents current and document how to inspect logs and
  force refresh after connectivity returns.
- IAM Roles Anywhere obtains credentials on demand and permits a configurable
  session duration. Align its profile duration with the IAM role maximum and
  alert before certificate expiry.
- Test reconnection timing for every node. Staggered credential recovery can
  cause pods to move from nodes that reconnect later to nodes that reconnect first.

## Review checklist

- Hybrid connectivity has redundant paths and end-to-end loss and latency alerts.
- Nodes carry physical zone labels and workloads span real failure domains.
- Required local services survive loss of the API server and regional dependencies.
- Pod eviction timing and split-brain behavior are tested for each application class.
- Host credential refresh and post-outage reconnection have a practiced runbook.
