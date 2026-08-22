# Amazon EKS Best Practices：Scalability

EKS scalability 是多維度的：control plane、nodes、cluster services、workloads、AWS quotas 與 external dependencies 必須以相容速度成長。大型 cluster 可降低 operational duplication，但也會增加 failure radius、upgrade complexity 與 tenant-isolation needs。

> 最後檢視：2026-08-20

## 保護 control plane 與擴展 data plane

- 限制突然的 node/pod bursts，為大型變更分階段並加入 jitter，避免 controllers、DaemonSets 與 clients 形成 thundering herd。
  - 大量 nodes 同時加入時，每個 DaemonSet、watch、image pull 與 registration 都會放大 control-plane 和 registry 負載；jitter 可將尖峰分散到較長時間。
- Cache `kubectl` discovery data，controllers 使用 shared informers 與 watches，對大型 list operations 分頁，避免頻繁 full-cluster polling。
- 監控 API Priority and Fairness、API latency、`429`、admission latency、scheduler backlog、controller-manager work，以及 etcd latency/size。
- 先在 client 修正過高 request rates；只有了解 traffic classes 與 consequences 後才調整 API Priority and Fairness。Cluster Autoscaler 僅在 scale 必要時 shard。
  - 提高 server limits 只會把壓力往 etcd 或 downstream controllers 移動；先使用 cache、watch、backoff 與 bounded concurrency 降低無效 requests。
- 使用 automatic node scaling 與多種合適 instance types；較大 nodes 降低 kubelet/DaemonSet/API overhead，較小 nodes 降低 failure blast radius，應依 workload density 與 recovery needs 選擇。
  - Node 越大，每個 node 的固定 overhead 比例越低，但單一 node failure 會同時移除更多 pods，replacement time 也可能更長。
- Right-size requests、控制 pod sprawl、監控 utilization 與 saturation，使用符合真實 demand 的 HPA v2 signals；自動化 AMI replacement 並考量 EBS attachment limits。
- 在 Linux nodes 使用 EKS Node Monitoring Agent，並評估 automatic node repair；規模增加後，人工判斷與替換 unhealthy nodes 的速度通常無法跟上 node churn。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/node-health.html)

## 擴展 services、workloads 並量測

- 水平擴展 CoreDNS、降低外部 lookups、測試後調整 `ndots`，並設定 readiness 與 lameduck timing。
- 依 nodes、pods、objects 與 events 數量為 Metrics Server、logging、monitoring、admission 與 policy agents sizing。
  - 這些 platform services 的負載通常隨 cluster objects 或 event rate 成長，不能只依 application CPU traffic 估算。
- 優先 EndpointSlices，限制每個 namespace 的 services、縮減 Deployment revision history，無需求時停用 automatic service links。
- 減少 dynamic admission webhooks，保持 logic 快速、可用且 scope 狹窄；高 pod density 可考慮 IPv6，並提前規劃 ELB、Route 53、Global Accelerator、CloudFront、storage 等 quotas。
- 定義 API request 與 pod startup latency 的 Kubernetes SLOs，在正常運作與 scaling events 期間量測 SLIs。
- 測試 churn rate，不只 maximum object count；比較 clusters 的 workload/platform behavior，及早讓 AWS support 參與 extra-large designs、quota increases 與 scale tests。
  - 相同 object count 下，大量 create/delete、rolling updates 或 node replacement 會產生更多 API writes、events 與 reconciliation work。

## Review checklist

- Control-plane clients 使用 watches、caches、backoff、pagination 與 bounded concurrency。
- Node diversity、density、storage limits 與 replacement rates 已 load-tested。
- CoreDNS、Metrics Server、observability 與 admission services 隨 cluster size 擴展。
- AWS quotas 與 IP capacity 包含 rollout/failure headroom。
- API latency、pod-start latency、scheduler backlog 與 throttling 有 alerts。

---
# Amazon EKS Best Practices: Scalability

EKS scalability is multi-dimensional: the control plane, nodes, cluster
services, workloads, AWS quotas, and external dependencies must all grow at
compatible rates. A large cluster can reduce operational duplication, but it
also increases failure radius, upgrade complexity, and tenant-isolation needs.

## Protect the control plane

- Limit sudden node and pod bursts. Stage large changes and add jitter so
  controllers, DaemonSets, and clients do not create a thundering herd.
- Cache `kubectl` discovery data, use shared informers and watches in
  controllers, paginate large list operations, and avoid frequent full-cluster
  polling.
- Monitor API Priority and Fairness, API request latency, `429` responses,
  admission latency, scheduler backlog, controller-manager work, and etcd
  latency and size.
- Fix high request rates at the client first. Adjust API Priority and Fairness
  only when traffic classes and consequences are understood.
- Shard Cluster Autoscaler only when its scale requires it, and make each shard
  responsible for a distinct set of node groups.

## Scale the data plane efficiently

- Use automatic node scaling and allow several suitable instance types so
  capacity can be fulfilled during demand spikes.
- Larger nodes reduce kubelet, DaemonSet, and API object overhead, while smaller
  nodes reduce failure blast radius. Choose from measured workload density and
  recovery needs.
- Keep node sizes similar within a node group when scheduling assumptions depend
  on a representative node template.
- Right-size requests, control pod sprawl, and monitor both utilization and
  saturation. Configure HPA v2 with signals that correspond to real demand.
- Automate AMI replacement. Account for EBS attachment limits, use multiple
  volumes when container I/O requires it, and avoid unnecessary node-local disk
  logging.

## Scale cluster services and workloads

- Scale CoreDNS horizontally, reduce avoidable external lookups, tune `ndots`
  only after testing resolution behavior, and configure readiness and lameduck
  timing for safe termination.
- Size Metrics Server, logging, monitoring, admission, and policy agents for the
  number of nodes, pods, objects, and events they process.
- Prefer EndpointSlices over legacy Endpoints, limit services per namespace,
  cap Deployment revision history, and disable automatic service links when
  workloads do not need them.
- Minimize dynamic admission webhooks per resource and keep webhook logic fast,
  available, and narrowly scoped.
- Consider IPv6 for high pod density and plan Elastic Load Balancing, Route 53,
  Global Accelerator, CloudFront, storage, and other AWS service quotas before
  they become bottlenecks.

## Measure scaling behavior

- Define Kubernetes service-level objectives for API request latency and pod
  startup latency, then measure their service-level indicators during normal
  operation and scaling events.
- Test churn rate, not just maximum object count. Node replacement, rapid
  rollouts, and pod creation can produce more control-plane load than a stable
  cluster of the same size.
- Compare workload and platform behavior across clusters to identify noisy
  controllers or configuration drift.
- Engage AWS support early for extra-large designs, quota increases, and scale
  tests that exceed the team's previous operating envelope.

## Review checklist

- Control-plane clients use watches, caches, backoff, pagination, and bounded concurrency.
- Node diversity, density, storage limits, and replacement rates are load-tested.
- CoreDNS, Metrics Server, observability, and admission services scale with cluster size.
- AWS quotas and IP capacity include rollout and failure headroom.
- API latency, pod-start latency, scheduler backlog, and throttling have alerts.
