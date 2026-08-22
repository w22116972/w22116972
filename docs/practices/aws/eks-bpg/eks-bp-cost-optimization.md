# Amazon EKS Best Practices：Cost Optimization

EKS cost optimization 是持續的 operating process，不是一次性的 instance discount。應量測 shared 與 workload costs、移除浪費、選擇合適的 capacity 與 storage，並確認節省成本不會削弱 availability 或 performance。

> 最後檢視：2026-08-20

## 建立 cost visibility

- 使用 Cloud Financial Management cycle：**See** usage 與 ownership、**Save** through optimization、**Plan** budgets 與 forecasts、**Run** 持續執行。
  - 先能將費用對應到 owner 與 workload，才有辦法判斷何處值得最佳化，並由負責團隊持續追蹤成效。
- 對 clusters 與 AWS resources 使用一致的 cost-allocation tags；使用 Cost Explorer、Split Cost Allocation Data for EKS，並以 CloudWatch Container Insights、Kubecost 或其他 in-cluster tool 補充。
- 有意識地分配 shared platform costs，盡可能按 team、namespace、application、environment 與 business value 單位報告。
- 設定 budgets 與 anomaly alerts；將 cost 與 utilization、saturation、service-level objectives 及 capacity risk 一併檢視。

## 最佳化 compute、network、storage 與 observability

- 依 production-like CPU、memory data、VPA recommendations、metrics 與 load tests right-size requests，不要把短暫 idle window 當作需求容量。
  - Requests 過高會留下無法排入其他 pods 的閒置容量；過低則可能造成 CPU throttling、OOM 或 node overcommit，節省成本時必須保留 tested headroom。
- 使用 HPA 與 scheduled scaling 降低 application consumption，並使用 EKS Auto Mode、Karpenter 或 Cluster Autoscaler 移除未使用的 node capacity。
- 以真實 requests、相容 instance shapes 與 consolidation 或 descheduling 改善 bin packing；使用 PDBs 與 failover headroom 保護 availability。
  - Bin packing 是把 pods 更緊密地放入較少 nodes；若 PDB 或剩餘容量不足，移除空閒 node 的過程可能中斷服務。
- 以 Spot、Savings Plans、reservations 與 On-Demand 分別對應 interruption tolerance、stable baseline 與 uncertain demand；只在 per-pod isolation 或 simplicity 值得時使用 Fargate。
  - Stable baseline 適合 commitment discount，波動部分保留 On-Demand 或 Spot，避免為不一定使用的 peak capacity 長期付費。
- 評估 Graviton 與 mixed-architecture cluster，但 container images 必須提供相容的 `arm64` build，並以 benchmark 驗證 price-performance；為避免 performance variance，同一 workload 通常固定使用單一 architecture。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-compute.html)
- 使用 EKS Auto Mode 時，確認 PDB、`karpenter.sh/do-not-disrupt`、過窄 NodePool constraints 或 topology rules 不會長期阻止 consolidation。
  - Auto Mode 只能在 pods 可安全移動且其他 nodes 有可行 placement 時移除或替換 underutilized nodes；阻塞 disruption 會讓閒置成本持續存在。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/auto-cost-control.html)
- 清理 completed jobs、unused resources、abandoned load balancers、unattached volumes、stale snapshots 及無 owner 或 purpose 的 resources。
- 在安全時讓 pod-to-pod 與 service traffic 留在 node 或 Availability Zone 內；使用 VPC endpoints、有效率的 ECR pulls 與合適 VPC connectivity 降低 NAT Gateway 與 cross-zone transfer。
- 依 workload evidence 選擇 EBS type、size、IOPS 與 throughput，並設定 snapshot、backup、EFS 與 FSx 的 retention 和 lifecycle policies。
- 只啟用所需 control-plane logs，設定 log retention、降低 debug output、控制 metric cardinality 與 scrape interval，並依 query 和 retention needs 選擇 trace sampling 與 storage tiers。
  - 高 cardinality labels 與過密 scrape 會同時增加 ingestion、storage 與 query cost；每個 signal 應能對應實際 alert、SLO 或 troubleshooting 用途。

## Review checklist

- 每項重要 shared 與 workload cost 都有 owner 與 allocation method。
- Requests 與 node capacity 以 observed demand 及 tested headroom 為依據。
- Pricing models 符合 interruption tolerance 與 baseline predictability。
- Cross-zone、NAT、registry、storage、logging、metrics 與 tracing costs 已量測。
- Savings changes 已依 availability、latency 與 recovery objectives 驗證。

---
# Amazon EKS Best Practices: Cost Optimization

EKS cost optimization is a continuous operating process, not a one-time instance
discount. Measure shared and workload costs, remove waste, select appropriate
capacity and storage, and verify that savings do not weaken availability or
performance.

## Establish cost visibility

- Use the Cloud Financial Management cycle: **See** usage and ownership,
  **Save** through optimization, **Plan** budgets and forecasts, and **Run** the
  process continuously.
- Apply consistent cost-allocation tags to clusters and AWS resources. Use Cost
  Explorer and Split Cost Allocation Data for EKS, then supplement them with
  CloudWatch Container Insights, Kubecost, or another in-cluster allocation tool.
- Allocate shared platform costs deliberately. Report cost by team, namespace,
  application, environment, and unit of business value where possible.
- Set budgets and anomaly alerts. Review cost alongside utilization, saturation,
  service-level objectives, and capacity risk rather than optimizing from the
  bill alone.

## Optimize compute

- Right-size workload requests from observed production-like CPU and memory
  data. Use VPA recommendations, metrics, and load tests; do not treat a short
  idle window as the required capacity.
- Use HPA and scheduled scaling to reduce application consumption, and use EKS
  Auto Mode, Karpenter, or Cluster Autoscaler to remove unused node capacity.
- Improve bin packing with realistic requests, compatible instance shapes, and
  consolidation or descheduling policies. Protect availability with PDBs and
  sufficient failover headroom.
- Diversify capacity. Use Spot for interruption-tolerant workloads, Savings
  Plans or reservations for stable baseline demand, and On-Demand for uncertain
  or non-interruptible demand.
- Use Fargate selectively when per-pod isolation or operational simplicity
  outweighs the price and scheduling limitations.
- Remove completed jobs, unused resources, abandoned load balancers, unattached
  volumes, stale snapshots, and other resources with no owner or purpose.

## Optimize network and storage

- Keep pod-to-pod and service traffic local to a node or Availability Zone when
  it is safe to do so. Account for the availability trade-off before enabling
  topology-aware or local traffic policies.
- Minimize NAT Gateway and cross-zone transfer by using VPC endpoints for AWS
  services, pulling from ECR efficiently, and choosing appropriate VPC
  connectivity. Measure data-processing and transfer charges before redesigning.
- Select EBS volume type, size, IOPS, and throughput from workload evidence, and
  review them over time. Apply snapshot and backup retention policies.
- Match EFS and FSx storage classes and deployment modes to access pattern,
  latency, throughput, durability, and lifecycle requirements.
- Keep container images small to reduce registry storage, transfer, and startup
  time.

## Optimize observability

- Enable only the control-plane logs needed for operational and compliance
  goals, and move long-term logs to a lower-cost store when appropriate.
- Set explicit log retention, reduce unnecessary debug output, and filter noisy
  records before ingestion.
- Collect metrics that drive decisions. Control high-cardinality labels and
  choose a useful scrape interval instead of collecting every possible series.
- Apply trace sampling, including tail sampling when rare failures must be
  retained, and choose storage tiers based on query and retention needs.

## Review checklist

- Every material shared and workload cost has an owner and allocation method.
- Resource requests and node capacity are based on observed demand and tested headroom.
- Pricing models match interruption tolerance and baseline predictability.
- Cross-zone, NAT, registry, storage, logging, metrics, and tracing costs are measured.
- Savings changes are validated against availability, latency, and recovery objectives.
