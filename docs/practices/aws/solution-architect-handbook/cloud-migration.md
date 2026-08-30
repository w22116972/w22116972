---

---

# Cloud Migration and Cloud Architecture Design

> Source: `aws/references/solution-architect-handbook/cloud-migration.pdf`

本章說明 cloud deployment model、cloud-native architecture、migration strategy、hybrid cloud 與 CloudOps。重點是：cloud migration 不是單純搬 server，而是根據 business objective、application readiness、risk、cost 與 modernization value 選擇合適策略。

## Public, Private, Hybrid, Multi-Cloud

- Public cloud：由 AWS、Azure、GCP 等 provider 提供 compute、storage、network、database、security、analytics 等 service，透過 internet/API 使用，通常採 OpEx/pay-as-you-go。
- Private cloud：組織自行建置與管理 cloud-like infrastructure，控制權高，但彈性、global reach 與 service breadth 通常不如 public cloud。
- Hybrid cloud：on-premises/private cloud 與 public cloud 並存，常見於大型企業漸進 migration、資料主權、低延遲或 legacy dependency 場景。
- Multi-cloud：同時使用多個 public cloud provider，可降低 provider lock-in、使用不同 provider 優勢，但會增加 governance、networking、security、operation complexity。

Public cloud 的運作模型是由 provider 將 compute、network、storage、database、security 與 managed platform services 以 API 形式提供給 customer。Customer 不需要採購與維護 physical infrastructure，但仍要負責 workload architecture、identity、data protection、configuration、monitoring 與 governance。

Hybrid cloud 在 migration project 特別常見，因為 enterprise application 很少能一次全部搬遷。部分系統可能因 mainframe、特殊硬體、vendor license、法規、資料主權、low-latency dependency 或 lifecycle constraints 留在 on-premises。這時 cloud 架構要先接受「長期混合」的現實，而不是假設 hybrid 只是短暫過渡狀態。

Multi-cloud 的價值通常來自 negotiation leverage、specific service capability、resilience 或 regulatory requirement，但它不是免費的彈性。每多一個 provider，就多一套 IAM、networking、observability、billing、IaC、security control 與 team skill requirement。只有當 business benefit 大於管理複雜度時才應採用。

## Public Cloud Architecture

Public cloud 是高度 virtualized、透過 internet 存取的 shared infrastructure。其核心價值：

- 快速 provisioning，不需長時間採購硬體
- Global infrastructure 可支援全球部署
- High availability 與 durability 可透過 multi-AZ、multi-region、replication 達成
- IaaS、PaaS、SaaS、FaaS 提供不同抽象層級
- Shared responsibility model：provider 負責 cloud 的安全，customer 負責 cloud 裡 workload 的安全

Public cloud storage 透過 replication model 提供 high durability and availability，例如 object storage 會在多個 facility 保存 copies。Compute 層則可透過 multiple Availability Zones、load balancer、Auto Scaling 與 health checks 移除 unhealthy nodes。架構師不能只把這些能力列為服務特色，必須把它們放進 workload design：資料是否跨 AZ、application 是否 stateless、database 是否支援 failover、DNS/routing 是否可切換、backup 是否能 restore。

Public cloud architecture 也讓 application 可以更接近全球使用者。Global users 可透過 region selection、edge location、CDN、DNS latency routing 與 local data residency design 取得更好的 latency 與合規性。但 multi-region 會增加 data consistency、deployment coordination、cost allocation 與 incident response 複雜度。

## Cloud-Native Architecture

Cloud-native 不只是把 application 放到 cloud VM 上，而是利用 cloud services、automation、managed service、serverless、distributed design 與 elasticity 重新設計 workload。

主要特性：

- On-demand scale
- Distributed architecture
- Managed service 優先
- Automation for recovery/scaling/deployment
- Global reach
- Cost optimization through elasticity
- Faster innovation

例如 micro-blogging application 可用 API Gateway、AWS Lambda、managed database、object storage、CDN 等服務，讓 scalability、HA、DR、operation 由平台能力支援。

PDF 中的 cloud-native example 類似 micro-blogging workload：前端透過 CDN/edge delivery 服務 static content，API Gateway 接受 request，Lambda 處理 business logic，DynamoDB 作 NoSQL data store，S3 保存 media/object，並用 managed service 處理 authentication、monitoring 與 event-driven integration。這種設計把 undifferentiated heavy lifting 交給 cloud provider，team 專注在產品功能。

Cloud-native migration 不一定要一次完成。常見方法是先把某些模組 containerize 或 serverless 化，例如 image processing、notification、report generation、search index update，再逐步拆出 domain service。這能降低一次性重寫風險，同時累積 cloud operating model。

## Cloud Migration Drivers

常見 migration trigger：

- Hardware 或 software end-of-life
- Data center contract 到期
- 需要 global expansion
- 需要 elasticity 與 cost optimization
- 需要 modernize legacy systems
- Security/compliance 改善
- 需要更快 delivery 與 innovation

Migration strategy 必須由 business objective 驅動，而不是單純從技術偏好出發。

Cloud migration initiative 常由明確事件觸發：data center lease 到期、hardware refresh cost 太高、security audit 要求改善、system performance 無法支援 demand、global expansion 需要新 region、或 legacy platform skill 越來越少。不同 trigger 會導致不同 migration priority。Data center exit 通常偏向 rehost/relocate；innovation 與 agility 通常偏向 refactor/rearchitect；非差異化 business function 則可能 repurchase SaaS。

組織層面也要處理 people/process/tooling。Cloud adoption 不只是選 AWS service，還要建立 landing zone、account strategy、network connectivity、identity federation、tagging/billing policy、security baseline、CI/CD、monitoring、incident response 與 team training。

## Migration Strategies

### Lift and Shift / Rehost

Rehost 是快速把 existing server/application 搬到 cloud，通常變更最少。適合 packaged software、deadline 壓力大、data center exit、或先搬後優化。

優點是 fast、predictable、repeatable、low upfront engineering。缺點是無法立即取得 cloud-native 最大價值。

Rehost 適合 packaged application、vendor software、低變更容忍度系統，或需要快速離開 data center 的 migration wave。通常會先盤點 VM、OS、network ports、storage volumes、dependencies 與 performance baseline，再用 migration tooling 搬遷。搬上 cloud 後仍可逐步做 right-sizing、backup automation、monitoring 與 managed database replacement。

### Replatform

Replatform 是在 migration 過程中做有限調整，例如 OS upgrade、database version upgrade、改用 managed database。比 rehost 多一點工程投入，但能改善 maintainability 與部分 operation burden。

Replatform 的原則是「不重寫 business logic，但替換部分 platform」。例如 application server 仍維持原架構，但 database 從 self-managed MySQL 移到 Amazon RDS；static files 從 local disk 移到 S3；batch server 改成 managed scheduler 或 container task。它通常能在可控風險下取得 cloud benefit。

### Relocate

Relocate 常見於 VMware 或 container workload，將整個 runtime/platform 搬到 cloud 對應環境，developer 投入相對低，適合希望快速移動現有平台的企業。

Relocate 特別適合已經標準化在 VMware 或 Kubernetes/container platform 的組織。它的目標是搬動 hosting substrate，而不是改 application code。好處是 migration schedule 較短、testing scope 較小；限制是 application architecture 仍保留原本的 coupling、statefulness 與 scaling limitation。

### Refactor / Rearchitect

Refactor 是重寫或大幅改造 application，使其符合 cloud-native model，例如 microservices、serverless、containerization、event-driven architecture。

優點是長期 cost、scale、agility、resilience 收益高；缺點是 upfront effort、risk、testing 成本高。

常見 refactor 範例包含：

- 將 monolith 拆成 microservices 或 modular services。
- 將 batch/reporting 改成 event-driven 或 serverless processing。
- 將 session state 從 server 移到 distributed cache/database。
- 將 synchronous integration 改成 queue/event based integration。
- 將 relational-only workload 拆出 search、NoSQL、object storage 或 analytics store。

Refactor 前要用 business value 排序，不應只因技術偏好而重寫。優先處理會限制 scale、release velocity、security posture 或 cost efficiency 的模組。

### Repurchase

以 SaaS 替代既有 application，例如用 cloud CRM、HR、ERP solution 取代自建系統。適合非核心差異化能力，能降低 maintenance responsibility。

Repurchase 要評估 data migration、identity integration、process change、customization loss、license/subscription cost 與 vendor lock-in。它不等於沒有架構工作；仍要設計 SSO、data export/import、integration API、audit、backup/retention 與 business continuity。

### Retain

保留在現有環境，通常因為 regulatory、latency、dependency、technical incompatibility 或 business priority 不足。

Retain 的 workload 不代表永遠不動，而是目前不應進入 migration wave。要在 portfolio 裡記錄保留原因、review date、dependency、risk 與 future migration trigger。否則 retained system 會成為 undocumented legacy island。

### Retire

關閉不再需要的 application，直接降低 cost、security risk 與 operational burden。

Retire 通常是 migration assessment 中最直接的節省來源。許多 enterprise portfolio 會存在 owner 不清、使用量極低、功能被其他系統取代、只為歷史報表存在的 application。Retirement 前要確認資料保存、legal retention、user communication、dependency removal 與 DNS/job/integration cleanup。

## Migration Planning

Cloud migration 應先做 workload discovery 與 analysis：

- Application inventory
- Dependency mapping
- Business criticality
- Data classification
- Security/compliance requirement
- Current cost/TCO
- Performance baseline
- Migration wave planning

之後建立 migration plan，包含 application migration、data migration、server migration、integration、validation、cutover 與 rollback。

Discovery 階段要收集：

- Server inventory：CPU、memory、disk、OS、middleware、runtime version。
- Application inventory：business owner、technical owner、SLA、users、release cycle。
- Dependency map：database、file share、message queue、LDAP/AD、third-party API、batch jobs。
- Network profile：ports、protocols、firewall rules、latency-sensitive flows。
- Data profile：volume、growth、classification、retention、RPO/RTO。
- Performance baseline：peak/average utilization、transaction volume、latency。
- Operational profile：backup、monitoring、patching、incident history。

Analysis 階段把 discovery data 轉成 migration decision。常見輸出包含 application grouping、wave plan、migration strategy per workload、risk list、dependency resolution、landing zone requirement、test plan、rollback plan 與 budget forecast。

Design 階段要針對 target cloud architecture 補齊 foundation：account/VPC/subnet/security group/IAM/KMS/logging/monitoring/backup/tagging/CI-CD。Application design 還要處理 stateless conversion、session storage、object storage、database choice、serverless/container suitability、autoscaling 與 high availability。

Execution 階段通常分成 application migration、data migration、server migration、integration/validation、cutover。Data migration 可使用 network transfer、database replication、bulk import、offline transfer device 或 dual-write/CDC。Server migration 要確認 image、driver、license、hostname/IP/DNS、security agent、monitoring agent 與 startup order。

Validation 不只做 smoke test。應包含 functional test、performance comparison、security validation、backup/restore test、integration test、data reconciliation、user acceptance test 與 operational readiness review。Cutover 前要有 freeze window、rollback criteria、communication plan 與 sign-off。

## Hybrid Cloud Architecture

Hybrid cloud 常見於：

- Legacy application 仍留在 on-premises
- 部分資料因合規或 latency 留在本地
- Cloud 作為 DR site
- 部分 workload 先 migration，其他逐步 modernize

設計重點：

- Stable network connectivity，例如 VPN、Direct Connect
- Identity federation
- Consistent security controls
- Centralized monitoring/logging
- Data synchronization
- Clear ownership between on-premises and cloud teams

Hybrid connectivity 要先設計可靠性與安全性。VPN 適合較快建立或備援，Direct Connect 適合 predictable bandwidth、低 jitter、private connectivity。通常會搭配 redundant links、BGP routing、segmented VPC/VLAN、central inspection、DNS resolution plan 與 identity federation。

Hybrid data architecture 常見挑戰是資料同步方向、conflict resolution、latency、schema change 與 cutover timing。若 on-premises application 與 cloud service 同時讀寫資料，必須明確定義 system of record。否則 migration 中會產生資料不一致。

Hybrid operation 也要統一 monitoring 和 incident process。CloudWatch、third-party APM、SIEM、CMDB、ITSM、ticketing/runbook 應該能跨 on-premises 與 cloud workload 建立共同視圖。

## CloudOps

CloudOps 是將 cloud workload operationalize 的方法，重點是透過 automation、monitoring、governance、cost control 與 continuous improvement 管理 cloud lifecycle。

Cloud migration 成功的衡量不只是 workload 已搬到 cloud，而是搬遷後能穩定、可觀測、可治理、可優化。

CloudOps transition 要處理原本 ITIL/ITSM 流程與 cloud automation 的落差。Change management 不能仍以手動 ticket 為唯一控制點；應把 approved change 轉成 pipeline、policy-as-code、IaC review、automated test 與 audit trail。Operations team 要能處理 account governance、tagging compliance、cost anomaly、security finding、patching、backup、incident response 與 post-migration optimization。

Post-migration optimization 包含 right-sizing instance、調整 autoscaling policy、改用 managed service、移除 unused storage/IP/snapshot、導入 CDN/cache、改善 database index/query、建立 budget alarm 與 review reserved capacity/Savings Plans。

## Summary

Cloud migration 應以 business outcome 為核心。Rehost、replatform、relocate、refactor、repurchase、retain、retire 都是工具，沒有單一最佳策略。好的 migration plan 需要理解 workload dependency、data、security、cost、operation 與 modernization value，並在 public cloud、hybrid cloud、multi-cloud 之間做可落地的設計取捨。
