# Kubernetes 叢集擴展性與高可用性最佳實務

本文件整理大型 Kubernetes 叢集與跨可用區部署的設計原則。上游可擴展性數字是特定 Kubernetes 版本的測試邊界，不是雲端供應商的服務配額或每個工作負載的建議目標。

## 在擴展前定義邊界

- 依實際 Kubernetes 版本確認支援範圍；不要把其他版本的數字當成保證。
- Kubernetes v1.36 文件列出的上游邊界為 5,000 nodes、每個 node 110 Pods、總計 150,000 Pods 與 300,000 containers。
- 同時檢查雲端供應商的 instances、vCPU、IP、subnet、volume、load balancer、security rule、API rate limit 與 log stream 配額。
- 估算 Pod IP、Service IP、DNS query、EndpointSlice、API object、watch traffic、event 與 audit log 的成長。
- 以小批次擴展 nodes，設定停頓、健康檢查與 rollback gate，避免同時觸發大量 instance、network 與 API 建立請求。

## 擴展 control plane 與叢集服務

- 先垂直擴展 control-plane 元件，監控 CPU、memory、API latency、work queue、etcd latency 與 request rejection，再決定是否水平擴展。
- 每個 failure zone 至少部署一個 control-plane replica；重要環境使用至少三個 failure zones，並確保負載平衡器具備健康檢查。
- etcd 成員使用奇數 quorum、低延遲儲存與定期 snapshot；不要跨高延遲 region 建立單一 etcd quorum。
- Event 流量成為瓶頸時，可評估獨立的 etcd event storage，但必須一併設計備份、監控與故障復原。
- CoreDNS、metrics、networking、ingress、admission webhook、logging 與 monitoring 必須隨叢集規模擴展；保留 system-critical capacity，避免業務負載耗盡節點資源。
- 使用 ResourceQuota、LimitRange、API Priority and Fairness 與 admission policy 控制 noisy neighbor 及突發 API 流量。

## 跨可用區部署

- 將 control plane、worker capacity 與關鍵 add-ons 分散到至少三個 availability zones。
- 確認 nodes 具有正確的 `topology.kubernetes.io/zone` 與 `kubernetes.io/hostname` labels；缺少 topology key 的 node 不會正常參與相關 spread constraint。
- 不要只依賴 scheduler 的 soft defaults。對重要工作負載明確設定 `topologySpreadConstraints`，並確認 replica 數足以涵蓋目標 zones。
- 同時使用 PodDisruptionBudget、rolling update 策略、anti-affinity 或 spread constraints，並測試 node drain 與 zone failure。

```yaml
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payments
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          minDomains: 3
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: payments
```

`minDomains` 與其他欄位必須符合目標 Kubernetes 版本。若可用 zones 或 capacity 少於要求，`DoNotSchedule` 可能讓 Pod 保持 Pending；發布前必須測試降級情境。

## 儲存、網路與復原

- 對 zonal volumes 使用 CSI driver、`volumeBindingMode: WaitForFirstConsumer` 與合適的 `allowedTopologies`，讓 volume 與 Pod 在相容 zone 建立。
- 不要假設 zone-local volume 能在 zone outage 時立即跨區掛載。為 stateful workload 定義 replication、backup、restore、RPO 與 RTO。
- 驗證 load balancer、CNI、NAT、DNS、ingress 與 topology-aware routing 在 zone failure 時的行為、流量成本與 cross-zone latency。
- 保留跨 zones 的 spare capacity；node autoscaler 必須能在仍健康的 zones 建立替代 capacity。
- 復原流程不可依賴故障 zone 中的 controller、registry、DNS、Secret store 或唯一的 repair node。
- 定期執行 zone-loss、control-plane endpoint、node drain、DNS 與 storage restore 演練，並以應用程式 SLO 驗證結果。

## 檢查清單

- [ ] 版本邊界與所有 provider quotas 已確認並有告警。
- [ ] Node、Pod、IP、EndpointSlice、DNS、API 與 observability 容量均有預測。
- [ ] Control plane、etcd、worker 與 critical add-ons 跨至少三個 zones。
- [ ] 關鍵工作負載有明確 topology spread 與 disruption policy。
- [ ] Zonal storage、network path、autoscaling 與故障復原已實測。
- [ ] 擴展採分批、可觀測且可 rollback 的流程。

## 參考資料

- [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)
- [Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

---

# Kubernetes Cluster Scalability and High Availability Best Practices

This guide summarizes design principles for large Kubernetes clusters and multi-zone deployments. Upstream scalability numbers are version-specific tested boundaries, not cloud-provider quotas or recommended workload targets.

## Define boundaries before scaling

- Confirm limits for the Kubernetes version actually in use.
- The Kubernetes v1.36 page lists upstream boundaries of 5,000 nodes, 110 Pods per node, 150,000 total Pods, and 300,000 total containers.
- Check provider quotas for instances, vCPU, IP addresses, subnets, volumes, load balancers, security rules, API rates, and log streams.
- Forecast Pod and Service IPs, DNS queries, EndpointSlices, API objects, watches, Events, and audit logs.
- Scale nodes in observable batches with pauses, health gates, and rollback criteria.

## Scale the control plane and cluster services

- Scale control-plane components vertically first while monitoring resource use, API latency, queues, etcd latency, and rejected requests.
- Run at least one control-plane replica per failure zone. Important environments should use at least three zones and health-checked API load balancing.
- Use an odd-sized etcd quorum, low-latency storage, and tested snapshots. Do not stretch one etcd quorum across high-latency regions.
- Consider separate etcd storage for Events only when Event traffic is a proven bottleneck and the extra operational lifecycle is supported.
- Scale CoreDNS, metrics, networking, ingress, admission, logging, and monitoring with the cluster. Reserve capacity for system-critical workloads.
- Use ResourceQuota, LimitRange, API Priority and Fairness, and admission policy to control noisy neighbors and API bursts.

## Deploy across zones

- Distribute the control plane, workers, and critical add-ons across at least three availability zones.
- Verify `topology.kubernetes.io/zone` and `kubernetes.io/hostname` labels on nodes.
- Do not rely only on soft scheduler defaults. Define explicit `topologySpreadConstraints` for important workloads and provision enough replicas and capacity for the target zones.
- Combine spread rules with PodDisruptionBudgets and safe rollout strategies, then test node drains and zone failure.

Use the manifest above as a starting point. Confirm field availability for the target version. Strict `DoNotSchedule` constraints can leave Pods Pending when zones or capacity are insufficient, so test degraded operation.

## Storage, networking, and recovery

- For zonal volumes, use a CSI driver, `volumeBindingMode: WaitForFirstConsumer`, and appropriate `allowedTopologies`.
- Define replication, backup, restore, RPO, and RTO for stateful workloads; a zonal volume is not automatically cross-zone recoverable.
- Validate load-balancer, CNI, NAT, DNS, ingress, and topology-aware routing behavior during zone failure, including cross-zone cost and latency.
- Keep spare capacity across zones and ensure node autoscaling can replace capacity in healthy zones.
- Do not make recovery depend on a controller, registry, DNS service, Secret store, or repair node that exists only in the failed zone.
- Exercise zone loss, control-plane endpoint failure, node drains, DNS failure, and storage restore against application SLOs.

## Checklist

- [ ] Version limits and provider quotas are verified and monitored.
- [ ] Node, Pod, IP, EndpointSlice, DNS, API, and observability growth is forecast.
- [ ] Control plane, etcd, workers, and critical add-ons span at least three zones.
- [ ] Critical workloads define topology spread and disruption policies.
- [ ] Zonal storage, network paths, autoscaling, and recovery are tested.
- [ ] Scaling is batched, observable, and reversible.

## References

- [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)
- [Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
