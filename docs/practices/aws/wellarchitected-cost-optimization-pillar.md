# AWS Well-Architected Framework：成本最佳化支柱

成本最佳化（Cost Optimization）是在滿足 functional requirements、reliability 與 security 的前提下，以最低合理成本交付 business outcomes。重點不是壓低帳單，而是讓每筆 spend 都可歸屬、可解釋、可調整。

## 設計原則

- 建立 Cloud Financial Management（FinOps）能力，讓 finance、technology 與 business 共同對成本和 business value 負責。
- 採用 consumption model，依需求調整資源，並以 workload metrics 而非直覺做 right-sizing。
- 將完整成本、trade-offs 和 unit economics 納入 architecture 與 delivery decisions。
- 自動處理可重複的 lifecycle、scaling 與治理工作，但為每項 automation 設定 guardrails 與驗證。
- 持續 review usage、pricing、service release 與 business demand，將改善納入有 owner 的 backlog。

## Review map

| 問題群組 | Review 重點 | 應具備的證據 |
|---|---|---|
| COST01 | FinOps ownership、forecast、culture、business value | RACI、budget/forecast、KPI、review records |
| COST02-COST04 | Governance、allocation、lifecycle cleanup | policies、tags/Cost Categories、CUR、decommission records |
| COST05-COST08 | Architecture、right-sizing、pricing、data transfer | ADR、cost model、utilization、commitment、network-flow analysis |
| COST09-COST11 | Demand/supply、continuous review、automation | demand forecast、scaling evidence、review backlog、automation audit trail |

## 個別 best practices

### COST01 - Cloud Financial Management

#### COST01-BP01 - Establish ownership of cost optimization
指定 owner 負責成本治理；記錄 RACI、KPI、review cadence 與決策。

#### COST01-BP02 - Establish a partnership between finance and technology
讓 finance 與 engineering 共用成本、usage 與 business context；將 variance 轉為共同決策。

#### COST01-BP03 - Establish cloud budgets and forecasts
以 usage-based forecast 建立 budgets 並追蹤 accuracy；定期 review 而非只看月結。

#### COST01-BP04 - Implement cost awareness in your organizational processes
將成本納入 architecture review、procurement、change 和 release decisions。

#### COST01-BP05 - Report and notify on cost optimization
依 audience 提供可行動的 reports 與 alerts，說明趨勢、driver 與 owner。

#### COST01-BP06 - Monitor cost proactively
監看 anomaly、forecast、commitment coverage 與 unit cost；設定 investigation threshold。

#### COST01-BP07 - Keep up-to-date with new service releases
定期評估 AWS 新功能、pricing 與 managed service；記錄 trade-off 與採用決策。

#### COST01-BP08 - Create a cost-aware culture
讓 teams 看見自己的成本；用 education、feedback 與 shared KPI 建立 accountability。

#### COST01-BP09 - Quantify business value from cost optimization
將成本改善連到 revenue、customer outcome 或 delivery speed；區分降費與提高 unit economics 的投資。

### COST02 - Governance

#### COST02-BP01 - Develop policies based on your organization requirements
以 business、security、compliance 和 cost requirements 形成 versioned policies；用 controls 驗證遵循。

#### COST02-BP02 - Implement goals and targets
設定可量測且有 owner 的 cost/efficiency targets；依 priority 排序並定期 review。

#### COST02-BP03 - Implement an account structure
以 account 隔離 ownership、environment、workload 與 chargeback；避免 management account 承載一般工作。

#### COST02-BP04 - Implement groups and roles
用 least-privilege groups/roles 控制採購、budget、billing 與 resource changes；保留責任分工。

#### COST02-BP05 - Implement cost controls
對高風險 spend 使用 quotas、SCP、approved catalog 或 policy-as-code；先測試不會阻斷必要營運。

#### COST02-BP06 - Track project lifecycle
追蹤 project 從 proposal 到 retirement 的成本與 owner；到期時觸發 cleanup 與成本復核。

### COST03 - Monitor cost and usage

#### COST03-BP01 - Configure detailed information sources
啟用 CUR、Cost Explorer、tags 與必要 telemetry；定義粒度、retention 與 reconciliation 方法。

#### COST03-BP02 - Add organization information to cost and usage
以 tags、accounts、Cost Categories 將成本連到 owner、environment、application、customer 或 product。

#### COST03-BP03 - Identify cost attribution categories
定義一致的 allocation categories 與 shared-cost rules；公開處理未標記成本的流程。

#### COST03-BP04 - Establish organization metrics
建立 spend、forecast、coverage、utilization 與 unit-cost metrics 並固定定義。

#### COST03-BP05 - Configure billing and cost management tools
設定 Budgets、Cost Anomaly Detection、Cost Explorer 與 CUR；驗證 alerts 能到正確 owner 並被處理。

#### COST03-BP06 - Allocate costs based on workload metrics
將 shared platform cost 以 requests、transactions、active users 或其他 consumption drivers 分配。

### COST04 - Decommission resources

#### COST04-BP01 - Track resources over their lifetime
記錄 resource owner、purpose、creation date、lifecycle state 與 expiry；找出 idle/orphaned 資源。

#### COST04-BP02 - Implement a decommissioning process
定義 discovery、approval、data retention、dependency check、delete 與驗證步驟；保留 audit trail。

#### COST04-BP03 - Decommission resources
依已核准流程移除不再需要的 resources、snapshots、IPs、licenses 與 subscriptions；驗證 billing。

#### COST04-BP04 - Decommission resources automatically
對 ephemeral environments、expired snapshots 與已知 idle patterns 使用有 guardrail 的自動化；提供 notification 與 exception path。

#### COST04-BP05 - Enforce data retention policies
依 legal、security 與 business needs 自動套用 retention、archive 與 deletion policy。

### COST05 - Evaluate cost when selecting services

#### COST05-BP01 - Identify organization requirements for cost
將 cost target、risk tolerance、compliance、skills、time-to-market 與 customer requirements 寫入 decision criteria。

#### COST05-BP02 - Analyze all components of the workload
評估 compute、storage、network、data transfer、observability、support、license 與 operational labor 的完整成本。

#### COST05-BP03 - Perform a thorough analysis of each component
依 usage pattern、growth、availability、performance 與 operational model 比較 alternatives；保留 assumptions 與 sensitivity analysis。

#### COST05-BP04 - Select software with cost-effective licensing
比較 BYOL、subscription、open-source 與 managed alternatives 的 license、support、compliance 與 exit cost。

#### COST05-BP05 - Select components of this workload to optimize cost in line with organization priorities
依 business priority 決定優化順序；成本增加必須換得明確的 customer value、risk reduction 或 delivery speed。

#### COST05-BP06 - Perform cost analysis for different usage over time
對 steady、seasonal、bursty 與 growth scenarios 建模；避免用單一平均 usage。

### COST06 - Select resource type, size, and number

#### COST06-BP01 - Perform cost modeling
建立可重現 cost model，包含 demand、utilization、scaling、commitment、data transfer 與 failure-mode assumptions。

#### COST06-BP02 - Select resource type, size, and number based on data
依 CPU、memory、IO、network、queue、latency 與 availability data 選擇 resource；不用歷史規格或最大值。

#### COST06-BP03 - Select resource type, size, and number automatically based on metrics
以 autoscaling、serverless 或 scheduling 依 metrics 調整供給；驗證 scale-in 不傷害 SLO。

#### COST06-BP04 - Consider using shared resources
在 isolation、security、blast radius 與 chargeback 可接受時共用 platform、network 或 managed service；追蹤公平 allocation。

### COST07 - Select the best pricing model

#### COST07-BP01 - Perform pricing model analysis
比較各 pricing model 的 coverage、utilization、flexibility、term risk 與 break-even；建立在已驗證的 baseline 上。

#### COST07-BP02 - Choose Regions based on cost
在 latency、data residency、resilience 符合前提下，將 regional price 與 data-transfer impact 納入選擇。

#### COST07-BP03 - Select third-party agreements with cost-efficient terms
審查 marketplace 與 vendor contract 的 metering、minimum commit、renewal、support 與 exit clauses。

#### COST07-BP04 - Implement pricing models for all components of this workload
盤點所有可承諾的 compute、database、storage 與 software；持續修正 unused 或低-utilization commitments。

#### COST07-BP05 - Perform pricing model analysis at the management account level
在 organization 層級合併使用量與 commitments；平衡集中購買的折扣與各 team 的成本 accountability。

### COST08 - Plan for data transfer

#### COST08-BP01 - Perform data transfer modeling
量化 ingress/egress、cross-AZ、cross-Region、NAT、CDN 與 hybrid traffic；用代表性流量驗證模型。

#### COST08-BP02 - Select components to optimize data transfer cost
選擇能減少不必要 hops、集中 egress、cache 或在正確 location 處理資料的 components；檢查 failure domain。

#### COST08-BP03 - Implement services to reduce data transfer costs
採用 CloudFront、VPC endpoints、PrivateLink、local processing 或 compression；用 flow/log data 驗證節省。

### COST09 - Manage demand and supply resources

#### COST09-BP01 - Perform an analysis on the workload demand
分析 demand 的 baseline、peak、seasonality、predictability、growth 與 business criticality。

#### COST09-BP02 - Implement a buffer or throttle to manage demand
用 queue、rate limit、backpressure、reservation 或 graceful degradation 保護 expensive downstreams；定義 customer impact。

#### COST09-BP03 - Supply resources dynamically
以 autoscaling、scheduled scaling、serverless 或 batch scheduling 供應資源；量測 scale latency 與 SLO impact。

### COST10 - Optimize over time

#### COST10-BP01 - Develop a workload review process
建立固定 review cadence、participants、decision log 與 action owner；涵蓋 architecture、usage、pricing、lifecycle 與 unit economics。

#### COST10-BP02 - Review and analyze this workload regularly
比較 actual 與 forecast、baseline 與 post-change、cost 與 business outcome；只推進有明確預期 value 的改善。

### COST11 - Automating operations

#### COST11-BP01 - Perform automation for operations
將 tagging、shutdown、rightsizing、cleanup、scaling 與 reporting 轉成 versioned automation；量測成功率、節省額 與 side effects。

## Implementation and validation

1. 先建立 account/tag/Cost Category allocation，讓成本能連到 workload、owner 與 business driver。
2. 為每個重要 workload 建立 forecast、budget、unit-cost metric、anomaly response 與 review cadence。
3. 以實際 utilization、demand 和 SLO 選擇 resource、pricing model 與 scaling strategy；變更前後都比較結果。
4. 對 lifecycle cleanup、scheduling、tagging 和 scaling 使用 versioned automation，先在受控範圍驗證再擴大。
5. 將改善項目記錄 owner、預期 business value、風險、截止日與驗證結果；未達預期時調整或停止。

## Checklist

- [ ] 每個 workload 都有成本 owner、allocation model、budget/forecast 與可行動的 alert。
- [ ] Cost、usage、unit economics 和 business outcome 可在相同 review 中解釋。
- [ ] Right-sizing、commitments、data transfer 與 idle resources 有定期 evidence-based review。
- [ ] Decommissioning 和 retention 有 owner、approval、audit trail 與自動化 guardrails。
- [ ] Cost optimization backlog 有可驗證的結果，而非只列出潛在節省。
