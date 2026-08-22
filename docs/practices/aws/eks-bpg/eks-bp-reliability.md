# Amazon EKS Best Practices：Reliability

可靠的 EKS platform 必須假設 pods、nodes、Availability Zones、network paths 與 external dependencies 都可能失敗。應設計 applications 與 platform services，使其能在預期 disruption 中持續運作，並以受控且可觀測的方式恢復。

> 最後檢視：2026-08-20

## Application、control plane 與 data plane reliability

- Production services 避免 singleton pods，使用多個 replicas，透過 topology spread constraints 或 pod anti-affinity 分散到 nodes 與 Availability Zones。
- 為 critical workloads 定義 PodDisruptionBudgets (PDBs)，依實際 availability requirement 設定，讓 maintenance 仍能前進。
  - PDB 太寬鬆無法保護 replicas，太嚴格則可能阻塞 node drain、upgrade 與 autoscaler scale-down；設定後必須實測 eviction。
  - PDB 只限制部分 API-initiated voluntary evictions；node crash、resource-pressure eviction、直接刪除 Pod/Deployment 不受其保護，scheduler preemption 也只會 best-effort 遵守 PDB。[Kubernetes 官方文件](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) [Kubernetes 官方文件](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
  - 因此 availability 仍須依靠足夠 replicas、topology distribution、capacity headroom 與 application-level recovery，不能只依賴 PDB。
- 設定準確 CPU/memory requests；使用 HPA 做 demand-driven replica scaling，以 VPA recommendations 改善 sizing，並測試與 node autoscaling 的互動。
- 慢啟動 applications 使用 startup probes，readiness probes 移除不可用 pods，僅在 restart 能恢復 health 時使用 liveness probes。
  - Startup probe 避免啟動中被誤殺，readiness 控制是否接收 traffic，liveness 才負責觸發 restart；三者不可用同一目的混設。
- 使用 rolling、canary 或 blue/green delivery，明確定義 rollback path，驗證 readiness、graceful termination、connection draining 與 data compatibility。
- 建立 application metrics、centralized logs、distributed traces、service-level indicators 與 user-visible alerts。
- 監控 API server、admission、etcd latency/size；讓 admission webhooks highly available、低 latency、狹窄 scope，並測試 dependencies 不可用時的 behavior。
  - Admission webhook 位於 API write path；若 timeout、scope 過廣或外部 dependency 失效，可能阻塞 deployments 甚至整個 cluster 的資源變更。
- 使用 resilient clients：cache reads、watch 而非 polling、exponential backoff，避免對 Kubernetes API 產生大型 bursts；維持支援中的 Kubernetes 與 add-on versions。
- 跨多個 Availability Zones 分散 worker capacity，保留替換 failed nodes 的 headroom，確認 quotas、subnet IPs、instance availability 與 autoscaling 能在各 zone 啟動 replacements。
  - Replica 分散本身不足以保證恢復；剩餘 zones 還必須有 IP、quota 與可取得的 instance capacity 承接 workload。
- Zonal EBS volumes 必須有相容的 zonal compute capacity 與 topology-aware scheduling；使用 EKS Auto Mode、managed node groups 或經測試的 Karpenter lifecycle。
- 在 Linux nodes 部署 EKS Node Monitoring Agent，並為 EKS Auto Mode、managed node groups 或 Karpenter 啟用或評估 automatic node repair，讓 fatal networking、storage、kernel、container runtime 與 accelerator failures 能觸發 repair。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/node-health.html)
- 以 requests、limits、PriorityClasses、quotas 與 QoS classes 保護 critical workloads；依需求擴展 CoreDNS、Metrics Server 與 NodeLocal DNSCache。

## Disruption、recovery 與 checklist

- 分別建模 voluntary disruptions、node loss、zonal loss、dependency failure 與 regional recovery。
  - 這些 failure modes 的偵測、影響範圍與復原方式不同，不應只用一次 node drain 測試代表所有情境。
- 評估 Amazon Application Recovery Controller (ARC) zonal shift 或 zonal autoshift，並先證明 cluster 能在少一個 Availability Zone 的情況下承載流量。
  - Nodes、application replicas 與 CoreDNS 必須跨 AZ 預先分散，剩餘 zones 也要有足夠 compute 與 IP capacity；否則啟動 zonal shift 本身就可能造成 outage。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/zone-shift.html)
- 定期測試 node drains、autoscaling、rollbacks、backups 與 restores；Chaos experiments 要有有限 blast radius 與可量測 success criterion。
- 明確定義 backup scope、retention、encryption、cross-region requirements、recovery time 與 recovery point objectives。
- 保持 runbooks、ownership、dashboards 與 escalation paths 在 outage 期間可用。
- Critical services 已跨 failure domains 部署；probes、shutdown handling、requests、autoscaling、upgrade、rollback、backup 與 restore 均有近期測試證據。

---
# Amazon EKS Best Practices: Reliability

Reliable EKS platforms assume that pods, nodes, Availability Zones, network
paths, and external dependencies can fail. Design applications and platform
services to continue operating during expected disruptions and to recover in a
controlled, observable way.

## Application reliability

- Avoid singleton pods for production services. Run multiple replicas and
  distribute them across nodes and Availability Zones with topology spread
  constraints or pod anti-affinity.
- Define PodDisruptionBudgets (PDBs) for critical workloads. Set them from the
  application's real availability requirement so maintenance can still make
  progress.
- Configure accurate CPU and memory requests. Use HPA for demand-driven replica
  scaling and VPA recommendations to improve sizing; test how both interact with
  node autoscaling.
- Use startup probes for slow-starting applications, readiness probes to remove
  unavailable pods from traffic, and liveness probes only when restarting the
  container can restore health.
- Use rolling, canary, or blue/green delivery with an explicit rollback path.
  Validate readiness, graceful termination, connection draining, and data
  compatibility during deployment.
- Instrument application metrics, centralized logs, and distributed traces.
  Define service-level indicators and alert on user-visible symptoms.

## Control-plane reliability

- Monitor API server request rate, latency, error codes, admission latency, and
  etcd request latency and size. A noisy controller or webhook can impair every
  workload even though AWS operates the control plane.
- Make admission webhooks highly available, keep their latency low, scope their
  rules narrowly, and choose failure policies intentionally. Test their behavior
  when the webhook or its dependencies are unavailable.
- Use resilient clients: cache reads, watch resources instead of polling, apply
  exponential backoff, and avoid large bursts against the Kubernetes API.
- Keep supported Kubernetes and add-on versions, test deprecations before an
  upgrade, and maintain a documented upgrade and recovery runbook.
- Choose cluster endpoint access and network paths that provide redundant,
  monitored connectivity for both nodes and operators.

## Data-plane reliability

- Spread worker capacity across multiple Availability Zones and retain enough
  headroom to replace failed nodes. Verify that quotas, subnet IPs, instance
  availability, and autoscaling policy can launch replacements in every zone.
- When workloads use zonal EBS volumes, ensure compatible compute capacity is
  available in the volume's zone and use topology-aware scheduling.
- Prefer EKS Auto Mode, managed node groups, or a tested Karpenter lifecycle over
  unmanaged, long-lived nodes. Monitor node health and replace unhealthy or
  outdated instances.
- Use requests, limits, PriorityClasses, quotas, and Quality of Service classes
  so critical workloads remain schedulable under contention.
- Scale CoreDNS and Metrics Server for cluster demand. Use NodeLocal DNSCache
  where appropriate and monitor DNS errors and latency.

## Disruption and recovery

- Model voluntary disruptions, node loss, zonal loss, dependency failure, and
  regional recovery separately. Each has a different mitigation.
- Regularly test node drains, autoscaling, rollbacks, backups, and restores.
  Chaos experiments should have a limited blast radius and a measurable success
  criterion.
- Make state recovery explicit: define backup scope, retention, encryption,
  cross-region requirements, recovery time, and recovery point objectives.
- Keep runbooks, ownership, dashboards, and escalation paths available during an
  outage, including when the primary cluster is inaccessible.

## Review checklist

- Critical services have multiple replicas across failure domains and a usable PDB.
- Probes, shutdown handling, requests, and autoscaling have been load-tested.
- The control plane, DNS, nodes, applications, and dependencies have actionable alerts.
- The cluster can add replacement capacity in every required Availability Zone.
- Upgrade, rollback, backup, and restore procedures have recent test evidence.
