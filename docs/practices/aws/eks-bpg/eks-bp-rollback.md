# Amazon EKS Best Practices：Rollback

Amazon EKS version rollback 可在文件規定的 seven-day window 內，將 in-place-upgraded control plane 恢復到前一個 minor version。它是 recovery safety net，不是 compatibility testing、backups、staged rollout 或 application-level recovery 的替代品。

> 最後檢視：2026-08-20

## 保持 rollback path 開放

- Non-Auto Mode clusters 先升級 control plane，並在 bake period 保留 N-1 nodes，避免 rollback 前必須立即反轉 data plane。
  - Kubernetes 支援有限的 version skew，因此舊版 nodes 可在 bake period 繼續運作，也讓 control-plane 問題能先被隔離觀察。
- 選擇同時相容 old/new Kubernetes minor versions 的 add-on versions，避免 unmanaged version drift。
- Bake period 避免只存在於新 version 的 APIs 或 features；rollback 前必須移除。
  - Rollback 後舊 control plane 無法理解新版本專屬資源或欄位，可能造成 reconciliation 失敗或 workload 無法管理。
- 依 Cluster Insights 檢查 `ROLLBACK_READINESS`，解決 `ERROR` findings；並獨立測試 self-managed add-ons、custom controllers、ingress、autoscaling、monitoring、policy systems 與 applications。
  - Cluster Insights 是 point-in-time 檢查且不涵蓋所有 third-party components，因此顯示 ready 仍不等於 application compatibility 已證明。
  - `--force` 只能選擇略過可略過的 insight checks，不能繞過 seven-day window、只能回復 N-1、cluster 必須由 in-place upgrade 而來，或 backward-incompatible EKS feature 等硬性限制。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/rollback-cluster.html)
- 確認 previous node images、add-on artifacts、manifests 與 deployment automation 仍可用且可信。
- 明確記錄 rollback scope：EKS 會恢復 API server、control-plane components 與 platform version，但不會回復 etcd data、customer workloads、EKS add-ons、persistent volumes/data，以及 managed、self-managed 或 hybrid nodes。
  - Auto Mode worker nodes 是例外，EKS 會先依 disruption controls 自動回復 Auto Mode nodes，再回復 control plane。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/rollback-cluster.html)

## 準備 data plane 並執行

- Managed node groups 在 control plane rollback 前恢復到 previous version，設定 workloads 與 PDBs 可承受的 update concurrency。
  - Concurrency 太高會同時移除過多 capacity，太低則可能超過 rollback timeout；應以 replica、PDB 與 replacement time 計算。
- Self-managed/hybrid nodes 先恢復相容 AMIs、packages 與 kubelet configuration；Fargate worker rollback 不支援，依文件替換受影響 pods。
- Auto Mode 檢查 NodePool disruption budgets、PDBs 與 `karpenter.sh/do-not-disrupt`；zero budget 或 blocked node 可能阻止進度。
- 透過支援的 EKS API、CLI 或 console workflow 明確 initiate rollback；CloudFormation stack rollback 不會自行執行 EKS control-plane version rollback。
  - CloudFormation 只會回復其 stack operation；EKS version rollback 是獨立的 service workflow，必須另外啟動與追蹤。
- 以 `DescribeUpdate` 追蹤 update、觀察 Cluster Insights、node versions 與 node-group status，並監控 application availability、scheduling、DNS、networking、storage、admission、authentication 與 service-level indicators。
- 將 Auto Mode `timeoutMinutes` 對齊 disruption policy 與 IaC timeout；完成後 reconcile IaC state，重新 deploy 或 revalidate upgrade 期間變更的 components。
  - 若 IaC 仍宣告新版本，下一次 apply 可能再次升級；rollback 完成不代表 source of truth 已一致。

## Review checklist

- Seven-day deadline 與 rollback restrictions 已記錄。
- Previous node、add-on、manifest 與 application versions 仍可 deploy。
- Readiness insights 與獨立 compatibility tests 沒有 blocker。
- PDBs 與 disruption controls 允許在 timeout 內替換 nodes。
- Monitoring 與 post-rollback application validation 有明確 owners。

---
# Amazon EKS Best Practices: Rollback

Amazon EKS version rollback can return an in-place-upgraded control plane to its
previous minor version within the documented seven-day window. It is a recovery
safety net, not a substitute for compatibility testing, backups, staged rollout,
or application-level recovery.

## Keep the rollback path open

- For non-Auto Mode clusters, upgrade the control plane first and retain N-1
  nodes during a bake period. This avoids immediately having to reverse the data
  plane before a control-plane rollback.
- Select add-on versions compatible with both the old and new Kubernetes minor
  versions. Prefer EKS managed add-ons when their rollback readiness checks are
  useful, and do not introduce unmanaged version drift.
- During the bake period, avoid new APIs or features that exist only in the new
  version. They must be removed before returning to the old control plane.
- Review rollback limitations for automatically upgraded and extended-support
  clusters before relying on rollback as the recovery strategy.

## Assess readiness

- Review the `ROLLBACK_READINESS` Cluster Insights immediately after upgrade and
  again before rollback. Resolve `ERROR` findings while the window remains open.
- Treat insights as point-in-time and incomplete. Independently test self-managed
  add-ons, custom controllers, ingress, autoscaling, monitoring, policy systems,
  and application compatibility.
- Confirm the previous node images, add-on artifacts, manifests, and deployment
  automation remain available and trusted.

## Prepare the data plane

- For managed node groups, return node groups to the previous version before the
  control plane. Apply update concurrency that the workloads and PDBs can sustain.
- For self-managed and hybrid nodes, restore compatible AMIs, packages, and
  kubelet configuration before control-plane rollback.
- Fargate worker rollback is not supported. Replace affected Fargate pods as the
  documented procedure requires; use force only with an understood version-skew risk.
- In Auto Mode, inspect NodePool disruption budgets, PDBs, and
  `karpenter.sh/do-not-disrupt` annotations. A zero disruption budget or blocked
  node can prevent rollback progress.

## Execute and monitor

- Initiate rollback explicitly through the supported EKS API, CLI, or console
  workflow. A CloudFormation stack rollback does not itself perform an EKS
  control-plane version rollback.
- Track the EKS update with `DescribeUpdate`, watch Cluster Insights, and monitor
  node versions and node-group status. Cluster status alone does not show every
  Auto Mode rollback phase.
- Monitor application availability, scheduling, DNS, networking, storage,
  admission, authentication, and service-level indicators throughout the change.
- Align Auto Mode `timeoutMinutes` with disruption policy and the timeout of the
  IaC system. Be prepared to adjust disruption budgets or cancel a stalled
  update through the supported operation.
- After rollback, reconcile IaC state and redeploy or revalidate every component
  that changed during the upgrade.

## Review checklist

- The seven-day deadline and rollback restrictions are recorded before upgrade.
- Previous node, add-on, manifest, and application versions remain deployable.
- Readiness insights and independent compatibility tests show no blocker.
- PDBs and disruption controls permit node replacement within the timeout.
- Monitoring and post-rollback application validation have named owners.
