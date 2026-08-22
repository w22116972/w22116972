# Amazon EKS Best Practices：Upgrades

EKS upgrade 應視為 control plane、data plane、add-ons、clients、manifests 與 applications 的 staged compatibility change。一次升級一個 minor version，保留 recovery path，並在進入下一階段前要求 evidence。

> 最後檢視：2026-08-20

## Change 前準備

- 追蹤 EKS release calendar，在 standard support 期間規劃 upgrades；檢視 Kubernetes change log、EKS release notes、deprecations 與 removed APIs。
- 使用 EKS Cluster Insights、`kubent` 或 Pluto 找出 deprecated APIs，更新 stored manifests 與 controllers，而不只是 deployment source。
  - 已存在於 cluster 的 objects 可能仍使用舊 API，即使 Git source 已更新；live state 與 source state 都必須掃描。
  - EKS Cluster Insights 通常每 24 小時 refresh，也可手動 refresh；deprecated API insight 使用 rolling 30-day observation window，因此 source 已修正不代表 finding 會立即消失。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)
- 檢查 VPC CNI、CoreDNS、`kube-proxy`、CSI drivers、AWS Load Balancer Controller、Metrics Server、autoscalers、service meshes、policy engines、observability agents 與 custom operators 的 compatibility。
- 檢查 node OS lifecycle；EKS 已於 2025-11-26 停止發布 EKS-optimized AL2 AMI，Kubernetes 1.32 是最後支援版本，升級至 1.33+ 前須完成 AL2023 或 Bottlerocket migration，並驗證 `cgroupv2`、bootstrap、JDK 與 drivers。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/eks-ami-deprecation-faqs.html)
- 確認 control-plane subnets 有 IP、EKS IAM role/KMS permissions 存在、service quotas 有 headroom，且能從預定 endpoint 存取 cluster。
- 啟用診斷所需 control-plane logs；backup cluster resources/state，記錄 restore steps，並在 maintenance window 前測試 backup。
- 在具代表性的 non-production environment 測試 target version，記錄 go/no-go criteria、owners、rollback conditions 與 communication paths。
  - 測試環境應包含相同 add-ons、policies、storage 與 traffic pattern；空白 cluster 通過不能證明 production workload 相容。

## 以受控階段升級

1. 確認 readiness checks、deprecation remediation、backup evidence 與 maintenance window。
2. 將 EKS control plane 升級一個 minor version。
3. 觀察 bake period，驗證 API operations、authentication、admission、scheduling、DNS、networking、storage 與 application SLIs。
   - Bake period 讓低頻或延遲出現的問題有時間被觀察；尚未符合 exit criteria 前不要繼續升級 data plane。
4. 將 managed add-ons 與 self-managed components 升級到支援新 control plane 的 versions，不要假設 cluster upgrade 會同步更新。
   - EKS Auto Mode 例外：VPC CNI、CoreDNS、`kube-proxy`、Karpenter、AWS Load Balancer Controller 與 EBS CSI 是 AWS-managed capabilities，不需也不應依 Standard EKS add-on workflow 手動升級；applications、self-managed controllers 與一般 EKS add-ons 仍由使用者負責。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/auto-upgrade.html)
5. 更新 `kubectl`、CI runners 與 administrative clients。
6. 依 controlled batches 升級 managed node groups、Karpenter nodes 與 self-managed nodes，遵守 version-skew rules。
7. Restart Fargate deployments，使新 pods 使用 current platform version。
8. 重跑 functional、performance、security、observability 與 recovery checks。

## 保護 workload availability

- 跨 nodes 與 Availability Zones 執行多個 replicas，設定 topology spread constraints 與不會阻塞所有 voluntary disruptions 的 PDBs。
- 正確使用 readiness、startup、liveness probes，實作 graceful termination，並給予 connection draining 足夠時間。
- 依 workload headroom 定義 node update concurrency；rollout 全程監控 Pending pods、failed evictions、unavailable replicas、volume attachment 與 load-balancer target health。
  - Update concurrency 必須同時受 available replicas、PDB、spare capacity 與 replacement time 約束，不能只追求最快完成。
- 使用 managed node groups、Karpenter drift/expiry 或 `eksctl` 等 automation 一致替換 nodes，避免長期留下 old nodes。

## 選擇 upgrade strategy

- 當 one-version sequence 與 rollback constraints 適合 workload 時，優先 in-place upgrades，以降低成本並保留 cluster identity。
- 跨多個 versions、變更 major architecture、需要 isolated validation 或逐步遷移 workloads 時，考慮 blue/green clusters；計入 duplicate cost、endpoints/OIDC、state movement、DNS/load-balancer cutover 與 rollback coordination。
  - Blue/green 提供獨立驗證與快速 traffic rollback，但 stateful data 若已在新環境寫入，仍需另外設計 data rollback 或 forward recovery。

## Review checklist

- Removed APIs 與 target-version compatibility 已從 live 與 source state 驗證。
- Add-ons、clients、nodes 與 Fargate workloads 都有明確 upgrade step。
- Backup/restore evidence 是最新的，rollback criteria 可量測。
- PDBs、topology、probes 與 capacity 允許 node replacement 完成。
- Post-upgrade checks 覆蓋 user traffic 與 effective cluster state，不只檢查 job success。

---
# Amazon EKS Best Practices: Upgrades

Treat an EKS upgrade as a staged compatibility change across the control plane,
data plane, add-ons, clients, manifests, and applications. Upgrade one minor
version at a time, preserve a recovery path, and require evidence before moving
to the next stage.

## Prepare before the change

- Track the EKS release calendar and plan upgrades during standard support.
  Review the Kubernetes change log, EKS release notes, deprecations, and removed
  APIs for both the current and target versions.
- Use EKS Cluster Insights and tools such as `kubent` or Pluto to find deprecated
  APIs. Update stored manifests and controllers, not only the deployment source.
- Check compatibility for VPC CNI, CoreDNS, `kube-proxy`, CSI drivers, AWS Load
  Balancer Controller, Metrics Server, autoscalers, service meshes, policy
  engines, observability agents, and custom operators.
- Verify control-plane subnets have available IP addresses, the EKS IAM role and
  KMS permissions still exist, service quotas have headroom, and cluster access
  works through the intended endpoint.
- Enable the control-plane logs needed for diagnosis. Back up cluster resources
  and state, document restore steps, and test the backup before the window.
- Test the target version in a representative non-production environment and
  record go/no-go criteria, owners, rollback conditions, and communication paths.

## Upgrade in controlled stages

1. Confirm readiness checks, deprecation remediation, backup evidence, and the
   maintenance window.
2. Upgrade the EKS control plane by one minor version.
3. Observe a bake period and validate API operations, authentication, admission,
   scheduling, DNS, networking, storage, and application service-level indicators.
4. Upgrade managed add-ons and self-managed components to versions supported by
   the new control plane. Do not assume the cluster upgrade updates them.
5. Update `kubectl`, CI runners, and administrative clients.
6. Upgrade managed node groups, Karpenter nodes, and self-managed nodes in
   controlled batches, respecting Kubernetes version-skew rules.
7. Restart Fargate deployments so new pods use the current platform version.
8. Re-run functional, performance, security, observability, and recovery checks.

## Protect workload availability

- Run multiple replicas across nodes and Availability Zones. Configure topology
  spread constraints and PDBs that preserve availability without blocking every
  voluntary disruption.
- Use readiness, startup, and liveness probes correctly; implement graceful
  termination and enough shutdown time for connection draining.
- Define node update concurrency from workload headroom. Monitor Pending pods,
  failed evictions, unavailable replicas, volume attachment, and load-balancer
  target health throughout the rollout.
- Use managed node groups, Karpenter drift and expiry, or automation such as
  `eksctl` to replace nodes consistently. Avoid leaving old nodes indefinitely.

## Choose an upgrade strategy

- Prefer in-place upgrades for lower cost and stable cluster identity when the
  one-version sequence and rollback constraints fit the workload.
- Consider blue/green clusters when crossing multiple versions, changing major
  architecture, requiring isolated validation, or migrating workloads gradually.
  Account for duplicate cost, new endpoints and OIDC identity, state movement,
  DNS and load-balancer cutover, and rollback coordination.

## Review checklist

- Removed APIs and target-version compatibility are verified from live and source state.
- Add-ons, clients, nodes, and Fargate workloads have an explicit upgrade step.
- Backup and restore evidence is current and rollback criteria are measurable.
- PDBs, topology, probes, and capacity allow node replacement to finish.
- Post-upgrade checks cover user traffic and effective cluster state, not only job success.
