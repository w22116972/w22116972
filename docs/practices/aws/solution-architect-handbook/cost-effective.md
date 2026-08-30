---

---

# Cost Considerations

> Source: `aws/references/solution-architect-handbook/cost-effective.pdf`

本章說明 cost optimization 的原則與技術。核心觀念：cost optimization 是 continuous process，不是一次性砍預算；目標是在不犧牲 business value、risk posture 與 required performance 的前提下，降低浪費並提升資源效率。

## Design Principles for Cost Optimization

Cost optimization 要從 architecture lifecycle 一開始就納入，而不是 production 後才補救。需要同時看：

- Business value
- Risk
- Total Cost of Ownership (TCO)
- Budget and forecast
- Demand management
- Service catalog
- Expenditure tracking
- Governance
- Sustainability / Green IT

PDF 強調 cost optimization 不能以犧牲 reliability、security 或 required performance 為代價。真正的 cost-effective architecture 是以最合理的成本達成 business outcome。這代表架構師必須在 design review 中把 cost 當成 first-class requirement，與 availability、latency、compliance、maintainability 一起討論。

成本治理要從 proof of concept 開始。POC 若沒有 tagging、budget、resource cleanup policy，很容易留下 idle environment。Production 前也要定義 owner、environment type、cost center、expected usage pattern 與 shutdown/schedule policy。

## Total Cost of Ownership

TCO 不只包含 upfront cost，也包含長期 operation、maintenance、training、support、retirement。

- CapEx：前期購置硬體、軟體、license 的 capital expenditure
- OpEx：日常 operation、subscription、maintenance、training、support cost

採購或建置 software 時應評估：

- Implementation cost
- Infrastructure cost
- License/subscription
- Integration cost
- Training
- Operation support
- Upgrade/patch
- Retirement/decommissioning

SaaS 可降低維運責任，但通常是 per-user subscription，需要評估使用者數量、使用頻率與 vendor lock-in。

TCO 評估也要把 hidden cost 列入，例如 migration effort、data transfer、integration middleware、security review、audit requirement、support contract、training、administrator headcount、backup/DR、monitoring tool、logging storage、downtime impact 與 decommissioning cost。

Cloud TCO 的常見誤區是只比較 VM hourly rate 與 server purchase price。On-premises 還包含 data center space、power/cooling、network gear、hardware depreciation、procurement lead time、over-provisioning 與 hardware refresh。Cloud 則要注意 data egress、cross-AZ traffic、managed service premium、log ingestion/storage、snapshot growth 與 idle resources。

## Budget and Forecast

Budget 是長期計畫，forecast 是基於實際使用趨勢的估算。Cloud cost 變動快，forecast 能協助即時調整，避免等到 budget 已不可達成才處理。

做法：

- 建立 cost baseline
- 根據 growth/demand 預測 spend
- 針對 anomaly 建立 alerts
- 定期 review forecast vs actual
- 將 cost 與 business metric 連結，例如 cost per transaction/customer/order

Budget planning 應該分成 annual budget、monthly forecast、project-level estimate 與 workload-level guardrail。Forecast 要利用歷史 usage、planned launch、seasonality、marketing campaign、data growth 與 reserved capacity commitments。

AWS 上可用 AWS Budgets、Cost Explorer、Cost and Usage Report (CUR)、Anomaly Detection 與 Organizations consolidated billing。重點不是只產生報表，而是建立 action loop：誰收到 alert、何時 review、什麼 threshold 需要修正、是否要停用 resource 或調整 architecture。

## Demand and Service Catalog

Central IT team 應與 business units 收集 demand forecast，避免重複建置與資源過度配置。

可採兩種 approach：

- Centralized service catalog：提供標準化 service，方便治理與成本控制
- Business-unit autonomy with guardrails：給團隊彈性，但透過 policy、tagging、budget、account structure 管理

Service catalog 的價值是避免每個 team 重新設計 VPC、database、CI/CD、monitoring、security baseline。平台團隊可提供 pre-approved patterns，例如 standard web app stack、serverless API stack、data pipeline stack、batch processing stack。這些 catalog item 應內建 tagging、logging、backup、encryption、IAM boundary 與 cost guardrail。

Demand management 要求 business unit 提供 growth assumption，例如 users、transactions、storage growth、peak season、SLA、reporting frequency。沒有 demand signal 時，engineering team 容易用過高規格設計，導致 idle capacity。

## Expenditure Tracking

成本需要可追蹤、可歸屬。常見機制：

- Show-back：讓 business unit 看到自己的使用成本
- Charge-back：將成本實際分攤到 business unit
- Budget alerts
- Resource tagging
- Account/OU structure
- Cost dashboard

通常先從 show-back 開始，建立 cost awareness，再逐步導入 charge-back。

Resource tagging 是 expenditure tracking 的核心。常見 tag 包含 application、environment、owner、cost-center、business-unit、data-classification、compliance、lifecycle、project。Tagging 必須在 provisioning pipeline 強制執行，不能依賴人工事後補標。

Account structure 也影響成本可視性。AWS Organizations 可依 business unit、environment、security boundary 或 workload 分帳，再用 consolidated billing 聚合。若所有 workload 都混在同一 account，後續 charge-back、權限隔離與 cost anomaly investigation 都會變困難。

## Continuous Cost Optimization

Cost optimization 應持續到「找出節省機會的成本高於可節省金額」為止。

常見措施：

- 關閉 idle resources
- Right-sizing compute/storage/database
- Auto Scaling / scheduled scaling
- Reserved Instances / Savings Plans
- Spot Instances for fault-tolerant workloads
- Storage lifecycle policies
- Cache 與 CDN 降低 backend cost
- Serverless pay-per-use
- 使用 managed service 降低 operation cost

Right-sizing 不是只把 instance 調小。需要觀察 CPU、memory、network、disk IOPS、database connections、queue depth、latency 與 error rate。調整後要驗證 user experience 沒有下降。對 predictable baseline workload 可用 Savings Plans/Reserved Instances，對 stateless、fault-tolerant、可中斷 batch workload 可用 Spot。

Storage optimization 要看 access pattern。Hot data 可放高效能 tier；warm/cold/archive data 應用 lifecycle policy 移到 cheaper storage class。Snapshot、old AMI、unused EBS volume、detached ENI、idle NAT gateway、old logs 常是被忽略的浪費。

Serverless 與 managed service 可以降低 idle cost 和 operation cost，但也要設計 concurrency limit、logging volume、payload size、retry storm 與 downstream capacity，否則 pay-per-use 會因事件量失控。

## Reducing Architectural Complexity

多個 business units 各自建相似系統，會造成 duplicated functionality、duplicated data、integration complexity 與 license waste。

降低成本的架構策略：

- 建立 reusable services
- API-based integration
- Modular architecture
- Centralized architecture governance
- 消除 duplicated capabilities
- 建立 enterprise platform team

Architectural complexity 會直接轉成成本：更多 integration、更多 duplicated data、更多 license、更多 monitoring、更多 incident surface。PDF 特別提到要消除 business units 之間重複建置的功能，建立共用 integration mechanism，讓新 application 可以透過 API 與既有 capability 互動。

Reusable service 不是把所有需求塞進一個巨型 shared platform，而是定義清楚 ownership、API contract、SLA、versioning 與 support model。否則共用服務會變成新的 bottleneck。

## Increasing IT Efficiency

提升 IT efficiency 的方法：

- Move suitable workload to cloud
- Automation for provisioning/deployment/operation
- Terminate unused systems
- Standardize tooling
- Measure productivity and business output
- 將 batch workload 設計成需要時啟動、完成後關閉

IT efficiency 的重點是把資源從 undifferentiated work 釋放出來。Cloud 可以讓 dev/test environment 在需要時建立，不需要時關閉；batch processing fleet 可在 job window 啟動，完成後 terminate；CI/CD 與 IaC 可減少人工 provisioning 錯誤。

效率評估要包含 lead time、deployment frequency、change failure rate、MTTR、environment provisioning time、incident count、manual ticket volume。若 cloud migration 後仍保留大量人工操作，成本通常只是從 hardware cost 轉成 labor cost。

## Standardization and Governance

Governance 可降低 overconsumption 與 misalignment。

Cloud 中常用方法：

- AWS Organizations / OU structure
- SCP / policy guardrails
- Resource tagging
- Standard naming convention
- Approved service catalog
- Cost allocation tags
- Budget and anomaly detection

Governance 不是阻止 team 使用 cloud，而是讓 team 在安全與成本邊界內快速交付。SCP、IAM permission boundary、config rule、policy-as-code、tag enforcement、approved AMI/container base image、network baseline、encryption default 都應自動化。

第三方 vendor 也要納入 governance。Vendor engagement model、license metric、support tier、data transfer、minimum commitment、renewal clause 都會影響 long-term cost。採購時應要求 vendor 能支援 financial goal，例如按使用量調整、提供 cost transparency、支援 cloud-native deployment 或 managed service integration。

## Green IT

Green IT 與 cost optimization 相關，因為降低能源消耗、減少 idle resources、提升 utilization 通常也降低成本。

可採措施：

- 使用 managed cloud infrastructure 提升 utilization
- 關閉不必要環境
- 選擇合適 region 與 instance type
- 延長硬體使用效率或降低硬體需求
- 以 automation 減少浪費

Green IT 與 cost optimization 的共同原則是提升 utilization 並降低 idle consumption。使用 serverless、autoscaling、container bin-packing、managed service、scheduled shutdown、right-sizing 都能同時降低成本與能源浪費。

AWS 提供 Customer Carbon Footprint Tool 類工具協助觀察 cloud workload 的 carbon impact。架構師在選擇 region、compute type、storage lifecycle 與 data retention policy 時，可把 sustainability 納入設計權衡。

PDF 的 AWS web application example 使用 scalable web tier、load balancing、Auto Scaling、managed database、object storage 與 monitoring/budget alert，目標是在高峰時擴容、低峰時縮容，並透過 cloud platform 提升 resource efficiency。

## Summary

Cost-effective architecture 不是最低成本架構，而是以合理成本達成 business outcome。核心工作是建立 TCO 視角、budget/forecast、show-back/charge-back、tagging/governance、continuous optimization、standardization 與 reusable architecture。成本必須與 performance、security、reliability、operation 一起平衡。
