---

---

# Principles of Solution Architecture Design

> Source: `aws/references/solution-architect-handbook/principles.pdf`

本章整理 Solution Architecture 設計時最常見、最核心的原則。重點不是列出所有可能的設計細節，而是建立一套判斷框架：如何在 scalability、availability、resiliency、performance、security、cost、operation 與 constraints 之間做合理 trade-off。

## 核心主題

- Building scalable architecture design
- Building a highly available and resilient architecture
- Design for performance
- Creating immutable architecture
- Think loose coupling
- Think service, not server
- Think data-driven design
- Adding security everywhere
- Making applications usable and accessible
- Building future-proof extendable architecture
- Ensuring architectural interoperability and portability
- Applying automation everywhere
- Design for operation
- Overcoming architectural constraints

## Scalable Architecture Design

Scalability 是系統能處理成長中 workload 的能力，可套用在 web layer、application layer、database layer 等不同層。雲端環境中更常強調 elasticity：需要時 scale out，需求下降時 scale in，以避免 idle resource 造成不必要成本。

### Horizontal Scaling

Horizontal scaling 是增加更多 server instance 來處理流量。例如原本 2 台 server 可處理 1,000 RPS，流量成長到 2,000 RPS 時，可增加到 4 台 server。

優點：

- 適合雲端與 distributed system
- 可搭配 Load Balancer 與 Auto Scaling
- 單一 instance failure 不應造成整體中斷

### Vertical Scaling

Vertical scaling 是增加同一台 server 的 CPU、memory、storage 等資源。這種方式常見於 relational database scaling，但成本通常會非線性上升，而且會遇到硬體上限。

設計時要注意：

- 超過一定門檻後 vertical scaling 成本高、彈性差
- Database 若成長到單機上限，需考慮 sharding
- 不應把 vertical scaling 當成長期唯一解法

### Predictive Scaling 與 Reactive Scaling

Predictive scaling 透過 historical data 預測流量，例如電商在節日、促銷日、週末或特定時段會有明顯流量變化，可提前增加 capacity。

Reactive scaling 則處理不可預期的突發流量，例如 flash sale 或外部事件造成的流量 spike。系統需要搭配 monitoring、scaling policy、cache、CDN 與 traffic offload 機制。

不論是哪種 scaling，都需要：

- 持續監控 application metrics
- 收集 traffic pattern
- 定義 scaling policy，例如 CPU utilization 超過門檻時增加 instance
- 測試 scale-out 過程是否會造成短暫 latency 或失敗

## Scaling Static Content

Static content 包含 image、video、CSS、JavaScript、static HTML page 等。這類內容通常 read-heavy，若直接放在 web server，會造成 storage 成長、latency 增加與 web tier load 過高。

建議做法：

- 使用 CDN，例如 Amazon CloudFront、Akamai、Azure CDN、Google CDN
- 使用 object storage，例如 Amazon S3
- 將 static content cache 到接近使用者的位置
- 讓 static storage 與 compute capacity 分開 scale

這樣可以降低 latency、減輕 web server 負載，並提升全球使用者體驗。

## Session Management for Application Server Scaling

Application layer 通常負責 business logic 與 database communication。若 user session 綁在特定 application server，horizontal scaling 或 instance replacement 可能導致使用者購物車、登入狀態或流程中斷。

設計原則：

- 將 user session 從 application server decouple
- 使用獨立 session store，例如 NoSQL database
- Application server 應盡量 stateless
- 在 application server 前方放 Load Balancer
- 搭配 Auto Scaling 自動增加或移除 instance

NoSQL database 如 Amazon DynamoDB 或 MongoDB 適合存放 semi-structured session data，也容易進行 horizontal scaling。

## Database Scaling

Relational database 提供 transactional consistency，但 horizontal scaling 較困難。設計上應先降低 primary database 負載，再考慮更複雜的 partitioning 或 sharding。

常見做法：

- 將 user session 放到 NoSQL database
- 將 static content 放到 object storage
- 使用 external cache，例如 Redis 或 Memcached
- Primary database 負責 write/update
- Read-heavy workload 使用 read replica
- 資料量持續成長時，透過 partition key 做 sharding

Read replica 可能有 milliseconds 級別 replication lag，架構設計與 application logic 需要考慮 eventual consistency 的影響。

## Elastic Architecture

Elasticity 的目標是 right-size architecture：高峰時有足夠 capacity，低峰時不浪費成本。

典型 AWS three-tier architecture 可包含：

- Amazon Route 53：高可用 DNS 與 routing
- Amazon CloudFront：CDN，分發 static content
- Elastic Load Balancing：分散流量到多個 target 與 Availability Zones
- Web layer Auto Scaling group：處理 dynamic content
- Application layer Auto Scaling group：執行 business logic
- Amazon RDS primary DB、read replica、standby instance：兼顧 read performance 與 failover

架構應能根據 demand 自動 expand/contract，同時保持 high availability。

## Highly Available and Resilient Architecture

High availability 的核心是避免 downtime。不同系統對 uptime 的要求不同：外部電商、社群平台可能需要極高 availability；內部 HR 或 intranet 可能可接受短暫 downtime。Availability 成本很高，Solutions Architect 必須根據 business criticality 避免 over-architecting。

Resiliency 則是系統在 failure 發生時仍能恢復並維持服務。它應套用到 infrastructure、application、database、security、networking 等所有 critical layer。

設計 resiliency 時需要：

- 定義 recovery time
- 找出 critical components
- 對必要 component 建立 redundancy
- 判斷何時 fix，何時 replace
- 讓系統具備 self-healing 能力

### Redundancy

Resilient architecture 需要 multi-layer redundancy：

- 同一 data center 內跨 rack
- 同一 region 內跨 Availability Zones
- 跨 geographic regions
- DNS-based routing 搭配 health check
- Database replication 與 automated failover

常見模式：

- CDN cache static content
- Load Balancer 將流量導向 healthy instances
- Auto Scaling 自動補足 server capacity
- Standby database 確保 database failure 時可 failover
- Region failure 時透過 DNS routing 將流量導到另一地點

### Component Failure

系統需要 health check 與 failure isolation。Shallow health check 只檢查 local host；deep health check 會檢查 dependencies，但成本與時間較高。

避免 cascading failure 的常見機制：

- Timeouts：避免 request 無限等待造成 resource exhaustion
- Traffic rejection：系統過載時拒絕新 request，保護現有流程
- Idempotent operations：重試不會造成重複扣款、重複建立資料等副作用
- Circuit breaker：偵測 failure pattern，暫停呼叫失敗服務，避免 failure 擴散

## Fault Tolerance

High availability 表示使用者仍可使用系統，但不代表 performance 不受影響。Fault tolerance 強調在故障發生時仍能維持 workload capacity。

例如原本需要 4 台 server 才能完整承載流量，若部署為兩個 data center 各 2 台，一個 data center failure 後仍可服務，但 capacity 只剩 50%。這是 high availability，但不是 100% fault tolerance。

100% fault tolerance 通常需要完整 redundancy，成本高。是否值得投入，取決於 application criticality：

- 電商平台：performance degradation 直接影響 revenue，可能需要高 fault tolerance
- 內部 payroll system：短時間 degraded performance 可能可接受

## Design for Performance

Performance 會直接影響 user engagement、conversion rate 與 revenue。設計時應在每一層考慮 performance，並透過 monitoring 持續改善。

重要原則：

- 針對真實 workload 做 load test
- 測試 Auto Scaling 生效前的 latency impact
- 根據 workload 選擇合適 compute、memory、storage
- Write-intensive workload 需注意 IOPS
- 慢網路環境可採 progressive loading，例如先載入文字，再載入圖片與影片

### Caching

Caching 是 performance design 的核心工具，可套用在多層：

- Browser cache：快取常用 web pages
- DNS cache：加速 domain lookup
- CDN cache：快取 image、video、static files
- Server memory cache：加速 application response
- Redis / Memcached：快取 frequent queries
- Database cache：將常用 query result 放在 memory

同時要設計 cache expiration 與 cache eviction，避免 stale data 或 memory pressure。

## Immutable Architecture

傳統 server 經過長期 patch、manual configuration 與特殊處理後，容易產生 configuration drift，導致 troubleshooting 困難，也讓 replacement 變得危險。

Immutable architecture 的核心是把 server 視為 replaceable resource，而不是需要長期照顧的固定資產，也就是常見的 “cattle, not pets”。

設計原則：

- Server 應可快速 provision、replace、dispose
- Application 應盡量 stateless
- 不 hardcode server IP 或 database DNS name
- 使用 Infrastructure as Code
- 不直接 patch live system
- 使用 golden image 建立一致 baseline
- 問題 server 下線前保留 logs 供 root cause analysis

這能提升 deployment speed、environment consistency、recovery speed 與 scalability。

## Think Loose Coupling

Tightly coupled architecture 中，component 彼此有直接依賴；任一 component failure 都可能造成連鎖錯誤，scale 或 replace server 也更困難。

Loose coupling 透過中介層與標準介面降低依賴：

- Load Balancer：將 request 導向 healthy application server
- Queue：讓 producer 與 consumer asynchronous communication
- Standard protocol：例如 RESTful API、message-based integration
- Microservices：每個 service 可獨立 evolve、scale、recover

Queue-based decoupling 很適合 image processing 等非同步工作：上傳 image 後，encoding、thumbnail、copyright check 等工作可由 worker 從 queue 平行處理。

Loose coupling 的價值：

- 降低 blast radius
- 提升 scalability
- 提升 availability 與 fault tolerance
- 更容易 integration
- 更容易單獨替換或升級 component

## Think Service, Not Server

Server-oriented thinking 容易導致硬體依賴與 tightly coupled architecture。Service-oriented thinking 則把功能拆成獨立 service，透過 standard protocol 溝通。

Microservice architecture 的特點：

- 每個 service 有明確 bounded function
- 可獨立部署、scale、維護
- 可使用自己的 framework 或 database
- failure blast radius 較小
- 適合 event-driven architecture

RESTful architecture 常使用 HTTP，payload 可為 JSON、XML 或 plain text。它 lightweight，適合建立 microservices。

Monolithic architecture 的問題是功能、部署與 database 緊密綁定；Microservices 則可讓 Login、Order、Cart、Payment 等服務依流量特性分別 scale。

但 microservices 也需要更細緻的 monitoring：既要看 individual service，也要看 end-to-end system health。

## Think Data-Driven Design

幾乎所有 software solution 都圍繞 data：收集、儲存、處理、保護與分析資料。電商系統有 customer、payment、order、inventory data；銀行系統則有 financial transaction data，且要求 integrity 與 consistency。

Data-driven design 的重點：

- 從 data flow 回推 architecture
- 根據 data type 選擇 storage
- 根據 consistency requirement 選擇 database model
- 根據 access pattern 設計 cache、index、replica、partition
- 收集 operational metrics 與 logs
- 用 dashboards、alerts、auto-healing 改善 operation
- 透過 data insight 提升 customer satisfaction 與 ROI

Data 不只影響 application design，也影響 operation、business decision 與 product strategy。

## Adding Security Everywhere

Security 是 solution design 的基本要求，任何缺口都可能造成 customer trust、reputation 與合規風險。設計一開始就要納入 security，而不是最後補上。

常見 compliance 與法規框架：

- PCI：保護 credit card information
- HIPAA：保護 healthcare patient data
- GDPR：保護 EU data privacy
- SOC：服務組織的 data management security controls

設計時要考慮：

- Physical security of data center
- Network security
- Identity and Access Management (IAM)
- Data security in transit
- Data security at rest
- Security monitoring
- Compliance logging 與 audit trail
- DDoS protection
- Security automation

Security 會影響 performance 與 latency，例如 encryption 需要額外 processing。架構設計要根據資料敏感度決定 encryption 與 control 強度。

DDoS 是 resiliency 的 security threat，可能讓合法使用者無法存取服務。應盡量把 workload 放在 private network，避免不必要的 public endpoint，並使用 automated detection、alerting 與 remediation。

## Usability and Accessibility

Feature-rich 不代表成功；使用者必須能有效使用系統。Usability 與 accessibility 直接影響 user experience 與 adoption。

### Usability

Usability 指使用者第一次使用時能多快理解 navigation logic、能否有效完成任務、出錯後能否快速恢復。

設計重點：

- 清楚直覺的 interface
- 簡單 navigation
- 高效 task completion
- User research
- Usability testing
- A/B testing
- 持續收集 feedback

### Accessibility

Accessibility 是讓不同能力、不同設備、不同網路條件的使用者都能使用系統。

設計可包含：

- Voice recognition
- Voice-based navigation
- Screen magnifier
- Read-aloud content
- 支援 vision/hearing impairment 的互動方式
- Localization，例如 Mandarin、Japanese、German、Hindi 等語言
- 適應 slow internet 與 old device

Solutions Architect 應與 Product Owner 一起理解 user limitation，並在設計階段納入 accessibility requirement。

## Future-Proof, Extendable and Reusable Architecture

Business 會持續變化，系統也需要加入新功能、整合新服務、支援更多 user base。因此 architecture 必須 extendable、flexible、reusable。

設計方式：

- 採用 loosely coupled architecture
- 使用 RESTful API 或 queue-based architecture
- 以 service module 建立 reusable capability
- 使用 API-based architecture 支援不同 channel
- 在 API framework 層考慮 OOAD，例如 inheritance、modularity

以電商為例，Product Catalog、Order、Payment、Shipping 可獨立成 service。Online app 與 store POS 都可重用 Payment service；未來 Gift Card、Food Service 或 Reward API 也可整合既有 service。

## Interoperability and Portability

### Interoperability

Interoperability 是 application 能透過 standard format 或 protocol 與其他系統協作。大型系統通常需要與 upstream/downstream systems 交換資料。

例子：

- E-commerce 需與 ERP、shipping、warehouse management、order management、transportation lifecycle management 整合
- Healthcare、manufacturing、telecom 也都有產業資料交換需求

設計時要：

- 找出 system dependencies
- 遵循產業 standard data exchange format
- 使用 JSON、XML、RESTful API 等 common format/protocol
- 減少額外 data transformation 成本

### Portability

Portability 是 application 能在不同 environment、OS、hardware 或 platform 上運作，且不需要或只需要少量修改。

設計時要注意：

- 技術更新速度快，系統需能適應新 language、platform、OS version
- Mobile app 需支援 iOS 與 Android
- 若要跨 OS，Java 這類 portability 高的 language 可能合適
- 若要跨 mobile platform，React Native 等 framework 可能合適

Interoperability 增強 extensibility；portability 增強 usability。若設計階段忽略，後續修正成本可能很高。

## Applying Automation Everywhere

Automation 能降低 human error、提升效率、節省成本，並讓團隊專注在更高價值問題。任何 repeatable task 都應評估是否可自動化。

可自動化項目：

- Application testing：自動化 regression test、production-scale test
- Rolling deployment：Canary testing、A/B testing
- IT infrastructure：Infrastructure as Code，例如 Ansible、Terraform、AWS CloudFormation
- Logging, monitoring, alerting：自動收集 metrics/logs，觸發 alerts 或 auto-scaling
- Deployment automation：CI/CD，支援小批量、頻繁 release
- Security automation：自動偵測 suspicious activity、traffic anomaly、security incident

Automation 也是 operational excellence 的基礎。

## Plan for Business Continuity

Disaster recovery 是 business continuity 的核心。當整個 region 或 data center 因 power outage、earthquake、flood、security attack 等事件失效時，企業仍需維持服務或快速恢復。

兩個重要指標：

- RTO (Recovery Time Objective)：可接受 downtime
- RPO (Recovery Point Objective)：可接受 data loss

RTO/RPO 越低，成本越高。是否需要 near-zero RTO/RPO，取決於 mission criticality。例如 stock trading application 不能遺失交易資料；railway signaling application 幾乎不能停機。

常見 disaster recovery strategy：

- Backup and Store：成本最低，RTO/RPO 最高；災難時從 machine image 與 database snapshot 還原
- Pilot Light：保留 machine image、小型 database 與 critical services，持續 sync data；災難時啟動與 scale up
- Warm Standby：DR site 持續以低 capacity 運行 application/database，災難時 scale up
- Multi-site：成本最高，near-zero RTO/RPO；多站點 active serving traffic

無論選哪種策略，都必須定期測試 failover，否則 DR plan 只是文件，不是能力。

## Design for Operation

Operational excellence 是長期服務品質的關鍵。架構設計必須包含 workload 如何 deployment、update、monitor、maintain 與 incident response。

設計重點：

- Logging
- Monitoring
- Alerting
- Deployment automation
- CI/CD pipeline
- A/B testing
- Blue-green deployment
- Security and compliance operation
- Small incremental changes
- Rollback strategy
- Runbook：例行操作文件
- Playbook：incident handling guide
- Root cause analysis：避免同類事件重複發生

Operation 不是一次性工作。每次 incident、failure、maintenance 都應轉化為改進系統的 feedback loop。

## Overcoming Architectural Constraints

Architecture 設計永遠受到 constraints 影響，包括 cost、time、budget、scope、schedule、resources、technology。Solutions Architect 的工作不是追求理論上最完美的設計，而是在限制下做合適 trade-off。

常見 trade-off：

- Performance vs cost：多層 caching 可提升 performance，但會增加成本與複雜度
- Scope vs speed：市場時機可能比完整功能更重要
- Availability vs budget：更高 redundancy 代表更高成本
- Security vs latency：encryption 與 security control 可能增加 processing overhead
- Modernization vs organizational constraints：大型組織中的技術變更需要考慮既有 system landscape

RESTful service 與 API wrapper 可協助整合 legacy system，例如 mainframe，讓舊系統能被新架構使用。

## MVP Approach

MVP (Minimum Viable Product) 是用最少必要功能滿足 early adopters，快速驗證產品方向並收集 feedback。

MVP 的目標：

- 立即提供 customer value
- 降低 development cost
- 盡快取得 feedback
- 透過 iteration 改善產品
- 避免在未驗證需求上浪費資源

### MoSCoW Prioritization

- Must have：沒有就不能 launch 的核心需求
- Should have：上線後高度期待的重要需求
- Could have：nice to have，沒有也不影響核心功能
- Won't have：本階段不做，使用者可能不會明顯感受到

MVP 應先交付 must-have requirement，讓 customer 取得可用產品，再根據 feedback 持續擴展。這能更有效運用 time、budget、scope 與 resources。

## Summary

Solution Architecture 的設計原則可歸納為幾個核心判斷：

- 系統要能 scale，也要能 elastic 地縮回來控制成本
- High availability、resiliency、fault tolerance 需要依 business criticality 決定投入程度
- Performance 需要跨 layer 設計，並透過 monitoring 與 load testing 驗證
- Infrastructure 應 immutable，server 應 replaceable
- Component 應 loose coupling，優先思考 service 而不是 server
- Data flow、security、usability、accessibility 都應在設計初期納入
- Architecture 要 future-proof、extendable、interoperable、portable
- Automation 與 operational excellence 是長期穩定運作的基礎
- Constraints 必須透過 trade-off 與 MVP approach 管理

好的架構不是堆疊最多技術，而是在明確限制下，用可驗證、可維運、可演進的方式滿足 business outcome。
