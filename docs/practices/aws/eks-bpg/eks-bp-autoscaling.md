# Amazon EKS Best Practices：Autoscaling

Autoscaling 是由多個 control loops 組成的鏈：workload autoscalers 建立或移除 pod demand，node autoscalers 再提供或釋放 capacity。可靠 scaling 需要準確的 pod requests、清楚的 scheduling rules、足夠的 AWS quotas，以及避免同時 disruption 的保護機制。

> 最後檢視：2026-08-20

## 選擇 node-scaling model

- 團隊希望 AWS 透過 Kubernetes APIs 管理更多 node、networking、load-balancing 與 storage lifecycle 時，使用 EKS Auto Mode。
  - 這會減少 platform team 維護底層元件的工作，但可調整的 infrastructure 細節也比自行管理少。
- 對多樣或快速變動、需要直接且具 scheduling awareness 的 instance selection 與 consolidation 的 workloads，使用 Karpenter。
  - Karpenter 會依 pending pods 的 requests 與 constraints 選擇 capacity，適合 node group 難以事先涵蓋的多樣需求。
- 當 node groups 與 Auto Scaling Groups 是既定 capacity boundary，或需要成熟功能時，使用 Cluster Autoscaler。
  - Cluster Autoscaler 調整既有 node groups 的 desired capacity，不會像 Karpenter 一樣逐次為 pod 選擇任意 instance type。
- 對可預測的 baseline demand，以及不能依賴 dynamic nodes 的 critical controllers，static managed node groups 仍然有用。

## 建立可靠的 scaling foundation

- 設定真實的 CPU、memory、ephemeral-storage 與 extended-resource requests；node autoscalers 只能依宣告的 pod requirements 做正確 placement。
- 使用代表 demand 的 HPA signals，例如 request rate、queue depth 或 concurrency；若使用 CPU，須理解 utilization target 是 CPU request 的百分比。
  - 例如 CPU request 為 `500m`、target 為 `80%`，平均使用量約 `400m` 就會觸發擴展；調低 request 也會同步降低絕對觸發門檻。
- 用 VPA recommendations 找出不良 requests，但自動 VPA action 必須與 HPA、PDBs 及 workload restart tolerance 協調。
- 在 scaling 前定義 topology spread、affinity、taints、tolerations 與 priorities，確認 constraints 留有可行 capacity choices。
- 為 subnet IP、EC2、vCPU、Spot、volume 及其他 service quotas 保留 headroom。

## Karpenter practices

- Production 固定使用經測試的 AMIs，並在 rollout 前於 non-production 驗證新 images。
  - 不應再把 EKS-optimized AL2 AMI 當成可持續更新的 production baseline；EKS 已於 2025-11-26 停止發布 AL2 AMI，Kubernetes 1.32 是最後支援版本，應遷移至 AL2023 或 Bottlerocket 並驗證 `cgroupv2` 與 application compatibility。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/eks-ami-deprecation-faqs.html)
- 將 Karpenter controller 執行於 Fargate 或 static managed node group，不要執行於其 lifecycle 由 Karpenter 控制的 nodes。
- 讓 NodePools 在 workload policy 允許下保持寬鬆；避免不必要的 instance type 限制，重疊 NodePools 應互斥或設定明確 weights。
  - 過窄 constraints 會降低可用 capacity 選項，造成 pending pods；重疊且無優先順序則可能讓 provisioning 結果難以預測。
- 使用多個 instance families、sizes 與 Availability Zones 改善 placement 與 Spot availability，並啟用 Spot interruption handling。
- 設定 NodePool resource limits 與 billing alarms，配置 consolidation 與 expiry；只在例外 workloads 使用 `karpenter.sh/do-not-disrupt`。
- 對非 CPU resources，在 consolidation 不得 overcommit 時對齊 requests 與 limits；必要時用 `LimitRange` 提供 namespace defaults。

## Cluster Autoscaler practices

- 執行與 cluster 相容的 Cluster Autoscaler version；可部署多個 replicas 提供 HA，但透過 leader election 同一時間只有一個 active replica 執行 scaling，不能靠增加 replicas 提高 throughput。IAM role 僅授予必要 scaling actions，並以 tags 限定 discovery。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/cas.html)
- 讓 node groups 盡可能少且具彈性；同一 group 的 nodes 應有等價 labels、taints 與 scheduling characteristics。
- 依實際 provisioning time 與 workload behavior 調整 scan interval、expander、scale-down thresholds 與 delay values。
- Spot groups 使用 scheduling shapes 相容的多種 instance types；若某類 capacity 應優先嘗試，使用 priority expander。
- 只有在 idle placeholder capacity 的成本值得換取較快 pod startup 時才使用 overprovisioning；僅因 scale constraints 已證實才 shard。
  - Overprovisioning 以低優先序 pods 預占 nodes，真實 workload 到來時再將它們驅逐，因此本質上是用持續成本換取較低 scale-out latency。

## Availability 與 validation

- 以 topology distribution 與真實 PDBs 保護 critical applications 與 controllers，確認 scale-down 仍能 drain nodes。
- Load-test 從 minimum capacity 的 scale-out、Spot interruption、zonal capacity shortage、consolidation，以及帶 stateful volumes 的 scale-down。
- 監控 unschedulable pods、provisioning latency、node readiness、disruption、failed launches、quota errors、unused capacity 與 scaling-loop health。
- 在 Linux nodes 使用 EKS Node Monitoring Agent，並為 EKS Auto Mode、managed node groups 或 Karpenter 評估 automatic node repair，讓 fatal node conditions 能觸發 reboot 或 replacement；此功能不支援 Fargate 與 Windows nodes。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/node-health.html)

## Review checklist

- Workload metrics 與 requests 反映真實 demand 與 resource use。
- Scheduling constraints 允許多個有效的 instance 與 zone choices。
- Autoscaler controllers 執行於 stable capacity，並使用 least-privileged IAM。
- Disruption、consolidation 與 PDB settings 允許安全持續前進。
- Scale-out、scale-down、interruption 與 capacity-shortage paths 均已測試。

---
# Amazon EKS Best Practices: Autoscaling

Autoscaling is a chain of control loops. Workload autoscalers create or remove
pod demand; node autoscalers then provide or release capacity. Reliable scaling
requires accurate pod requests, clear scheduling rules, sufficient AWS quotas,
and protection against simultaneous disruption.

## Choose the node-scaling model

- Use EKS Auto Mode when the team wants AWS to manage more of the node,
  networking, load-balancing, and storage lifecycle through Kubernetes APIs.
- Use Karpenter for diverse or rapidly changing workloads that benefit from
  direct, scheduling-aware instance selection and consolidation.
- Use Cluster Autoscaler when node groups and Auto Scaling Groups are the
  established capacity boundary or when its mature feature set is required.
- Static managed node groups can remain useful for predictable baseline demand
  and for critical controllers that must not depend on dynamically managed
  nodes.

## Build a sound scaling foundation

- Set realistic CPU, memory, ephemeral-storage, and extended-resource requests.
  Node autoscalers can only make correct placement decisions from declared pod
  requirements.
- Use HPA signals that represent demand, such as request rate, queue depth, or
  concurrency. When CPU is used, understand that its utilization target is a
  percentage of the CPU request.
- Use VPA recommendations to identify poor requests, but coordinate any
  automatic VPA action with HPA, PDBs, and workload restart tolerance.
- Define topology spread, affinity, taints, tolerations, and priorities before
  scaling. Confirm that constraints leave feasible capacity choices.
- Maintain subnet IP, EC2, vCPU, Spot, volume, and other service-quota headroom.

## Karpenter practices

- Pin tested AMIs in production and validate new images in non-production before
  rollout.
- Run the Karpenter controller on Fargate or a static managed node group, not on
  nodes whose lifecycle Karpenter controls.
- Keep NodePools as broad as workload policy permits. Avoid unnecessary instance
  type restrictions, and make overlapping NodePools mutually exclusive or give
  them explicit weights.
- Use multiple instance families, sizes, and Availability Zones to improve
  placement and Spot availability. Enable interruption handling for Spot.
- Set NodePool resource limits and billing alarms. Configure consolidation and
  expiry, but use `karpenter.sh/do-not-disrupt` only for exceptional workloads.
- For non-CPU resources, align requests and limits when consolidation must not
  overcommit them. Provide namespace defaults with `LimitRange` where helpful.

## Cluster Autoscaler practices

- Run a Cluster Autoscaler version compatible with the cluster and normally run
  one replica responsible for the cluster. Grant its IAM role only the required
  scaling actions and scope discovery with tags.
- Keep node groups as few and as flexible as practical. Nodes in one group
  should have equivalent labels, taints, and scheduling characteristics.
- Tune scan interval, expander, scale-down thresholds, and delay values from
  observed provisioning time and workload behavior.
- For Spot groups, use multiple instance types with compatible scheduling
  shapes. Use the priority expander when one class of capacity should be tried
  before another.
- Use overprovisioning only when the cost of idle placeholder capacity is
  justified by faster pod startup.
- Shard only for proven scale constraints; independent shards can provision
  duplicate capacity for the same unschedulable pod.

## Availability and validation

- Protect critical applications and controllers with topology distribution and
  realistic PDBs. Verify scale-down can still drain nodes.
- Load-test scale-out from minimum capacity, Spot interruption, zonal capacity
  shortage, consolidation, and scale-down with stateful volumes.
- Monitor unschedulable pods, provisioning latency, node readiness, disruption,
  failed launches, quota errors, unused capacity, and scaling-loop health.

## Review checklist

- Workload metrics and requests reflect real demand and resource use.
- Scheduling constraints allow multiple valid instance and zone choices.
- Autoscaler controllers run on stable capacity with least-privileged IAM.
- Disruption, consolidation, and PDB settings permit safe forward progress.
- Scale-out, scale-down, interruption, and capacity-shortage paths are tested.
