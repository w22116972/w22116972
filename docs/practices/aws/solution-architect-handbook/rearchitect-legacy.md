---

---

# Rearchitecting Legacy Systems

> Source: `aws/references/solution-architect-handbook/rearchitect-legacy.pdf`

本章說明 legacy system 的挑戰、modernization strategy、assessment、modernization techniques、cloud migration decision 與 mainframe modernization。核心觀念：legacy modernization 是 business risk、cost、技術債、skill、security、compliance 與 delivery risk 的綜合 trade-off。

## Legacy System Challenges

Legacy systems 通常已運作多年，可能仍支撐核心業務，但面臨：

- Difficulty keeping up with user demand
- High maintenance/update cost
- Proprietary software dependency
- Configuration drift
- Shortage of skills
- Poor or missing documentation
- Security vulnerabilities
- Compliance risk
- Incompatibility with modern systems
- Scalability limitations
- Expensive hardware infrastructure

這些問題會限制 business agility，也讓新技術如 analytics、ML、GenAI、IoT 更難整合。

Legacy system 的問題通常不是單一技術老舊，而是 business process、data model、integration、team knowledge 與 infrastructure 全部綁在一起。使用者可能已經習慣舊流程，但新需求如 mobile access、real-time analytics、API integration、global scale、security audit 會暴露原架構限制。

PDF 強調幾個常見痛點：

- User demand 成長時，legacy architecture 很難 horizontal scale。
- 老舊 OS/runtime/database 可能不再有 vendor support，patch 與 upgrade 風險高。
- Documentation 不完整，修改前無法確定 impact。
- 熟悉 COBOL、mainframe、proprietary middleware 的人才減少。
- Legacy system 往往依賴 custom point-to-point integration，與 modern API/event architecture 不相容。
- Security vulnerability 可能來自 EOL software、manual patching、弱 authentication、過度權限。

## Modernization Strategy

Modernization 可一次性大改，也可 phased approach。

- Big-bang modernization：一次替換大範圍系統，風險高、需要強 governance
- Phased modernization：逐 module 升級，風險較低，較容易保留 business continuity

優先順序應根據：

- Business criticality
- Security/compliance risk
- Cost saving potential
- Technical complexity
- Dependency count
- User pain
- Modernization value

Modernization strategy 要先連回 business outcome：降低維運成本、加快新功能交付、提升 scalability、改善 security/compliance、開放 API、支援 digital channel、改善 reliability。若沒有 business driver，modernization 很容易變成昂貴的技術重寫。

Phased modernization 常搭配 strangler fig pattern：在 legacy system 外逐步建立新 capability，將流量或功能一部分一部分導到新系統，直到舊模組可退休。這比 big-bang rewrite 更容易控制 risk，但需要 integration layer、data synchronization、routing 與 governance。

## Assessment

Modernization 前要深入 assessment：

- Application inventory
- Module dependency
- Code complexity
- Data dependency
- Integration points
- Infrastructure/runtime
- Security posture
- Compliance requirement
- Documentation state
- Skill availability
- Current TCO

目標是理解系統後再決定 rehost、replatform、refactor、rearchitect、replace。

Assessment 應產生 modernization heat map。每個 application/module 可依 business value、technical debt、security risk、dependency complexity、change frequency、operational cost、cloud readiness 評分。這能避免只改「看起來容易」的模組，而忽略真正限制 business 的核心瓶頸。

需要特別盤點：

- Upstream/downstream dependency：誰呼叫 legacy，legacy 呼叫誰。
- Shared database/table/file：哪些 application 共用資料。
- Batch schedule：nightly/monthly jobs 的順序與 SLA。
- Business rules hidden in code：是否只有程式碼保存規則。
- Non-functional requirement：latency、availability、RPO/RTO、audit。
- Data migration complexity：資料品質、編碼、schema、volume、retention。
- License and vendor constraint：cloud 是否可執行、license 是否允許。

## Modernization Approaches

### Encapsulation

在 legacy system 外建立 API wrapper，讓 modern systems 可透過 API 存取。變更小、風險低，但 legacy core 仍存在。

Encapsulation 適合需要快速開放 legacy capability 的場景，例如把 mainframe account lookup 包成 REST API。它可降低 integration friction，但不會降低 legacy maintenance cost 或技術債。若 wrapper 只是把舊系統複雜度原封不動暴露出去，長期仍會限制新系統。

### Rehosting

將 application 搬到新 infrastructure/cloud，程式改動少。適合快速 data center exit，但 modernization benefit 有限。

Rehosting 常用於 deadline-driven migration，例如 data center 關閉。優點是速度快、business change 小；缺點是 cloud cost 可能不理想，因為 workload 還保留固定 capacity、stateful server、manual operation。搬遷後應安排 optimization wave。

### Replatforming

升級 OS、database、runtime 或改用 managed service，例如 Windows Server 2012 EOL 後升級到 newer OS。比 rehost 多改動，但能降低維運風險。

Replatforming 的價值是用有限改動取得平台收益。例如把 self-managed database 移到 managed database、把 file storage 移到 managed shared filesystem、把 app runtime 升級到 supported version。它通常比 refactor 風險低，但仍需要 regression testing。

### Refactoring

改善 code structure，不一定改變外部行為。可逐步降低技術債。

Refactoring 可以改善 module boundary、消除 duplicated code、增加 automated tests、移除 dead code、整理 dependency。它常是 rearchitect 前的準備工作，因為沒有 test/documentation 的 legacy code 很難安全拆分。

### Rearchitecting

改變 system architecture，例如將 monolith module 轉為 service-oriented architecture 或 microservices。可重用部分 existing code，但需要重新設計 integration、data、deployment。

Rearchitecting 會改變 application decomposition、data ownership、communication pattern 與 deployment model。例：把 order、payment、inventory 從 monolith 拆出，改用 API/event integration。這需要處理 distributed transaction、eventual consistency、observability、service ownership 與 CI/CD。

### Redesigning and Replacing

完整重寫或以新產品/SaaS 替代。潛在收益高，但 cost、risk、change management 最大。

Redesigning/replacing 適合 legacy system 已無法支援 business strategy，或現成 SaaS/product 已能滿足非差異化需求。風險在於功能 parity、data migration、user training、business process change、integration rewrite。不要低估「舊系統裡隱藏的例外流程」。

## Cloud Migration Strategy for Legacy Systems

Cloud 可提供 elasticity、managed services、DR、security tooling、cost flexibility。但不是所有 legacy workload 都適合直接上 cloud。

Decision factors：

- Application still heavily used?
- Can it run on cloud-supported OS/runtime?
- Data sensitivity/residency?
- Dependency with on-prem systems?
- Latency requirement?
- TCO benefit?
- Modernization budget?
- Business tolerance for change?

建議先針對複雜 module 做 POC，驗證 migration feasibility、performance、integration、operation。

Cloud migration decision 要先判斷 legacy workload 是否能在 cloud supported OS/runtime 上運作。若 application 依賴特殊硬體、mainframe transaction monitor、old database driver、固定 IP/licensing dongle，就不能只用 lift-and-shift 假設。

POC 應挑選 high-risk module，而不是最簡單 module。目標是驗證最不確定的事項：performance、latency to on-prem dependency、data replication、authentication、batch window、monitoring、rollback。POC 結果應明確回答是否可 rehost/replatform，或必須 refactor/rearchitect。

Legacy cloud migration 也可以採混合策略：部分 module retain on-premises，部分 encapsulate with API，部分 replatform to managed database，部分 refactor to Lambda/API Gateway/DynamoDB。策略應針對 module，而不是整個 application 一刀切。

## Documentation and Support

Modernization 後要更新：

- Architecture document
- Runbook
- Playbook
- Dependency map
- API contract
- Data model
- Support ownership

否則新系統很快會變成下一代 legacy。

Documentation 不只是交付文件。它要支援 operation 與 future change，包含 architecture decision record、domain glossary、API spec、sequence diagram、data lineage、batch schedule、deployment runbook、DR runbook、monitoring dashboard、known limitations。Support model 要定義 L1/L2/L3 owner、SLA、escalation path、vendor dependency 與 release process。

## Mainframe Modernization

Mainframes 仍支撐許多金融、保險、政府核心 workload。Modernization 挑戰：

- Massive scale
- Mission-critical uptime
- COBOL/shared code dependency
- Complex batch jobs
- Data integrity
- Security/compliance
- Skill shortage

Cloud migration 可降低硬體成本並改善 elasticity，但必須 incremental、validated、risk-controlled。

策略：

- Migrating standalone applications
- Migrating applications with shared code
- Standalone API decoupling
- Refactor shared COBOL programs
- Gradual module migration

Mainframe modernization 常見最大問題是 shared code 與 shared data。Standalone application 較容易遷移：將程式碼轉換到 Java/.NET 等 runtime，資料與 batch 一起搬遷。Shared code application 則要先 decouple，否則移動一個 application 會破壞仍在 mainframe 的其他 application。

PDF 提到幾種 decoupling pattern：

- Standalone API：把 shared program 包成 API，讓 migrated application 與 mainframe application 都可透過 API 呼叫。
- Shared library：將 shared COBOL/program 轉成 Java/.NET library，打包給 migrated applications 使用。
- Message queue：用 queue/event 讓 migrated module 與 mainframe module asynchronous communication，適合不同 migration wave 需要長期共存。

Public cloud 對 mainframe modernization 的價值包含 elastic compute、managed database/storage、modern DevOps toolchain、API integration、analytics/ML integration、降低 hardware dependency。AWS Mainframe Modernization (M2) 提供 migration/modernization platform，可支援 refactor/replatform、managed runtime、development/testing/deployment。

## GenAI Assistance

GenAI 可作為 coding assistant，協助理解 legacy code、產生 documentation、轉換部分 code、撰寫 test。但不能取代 assessment、domain validation、security review 與 production testing。

GenAI 可協助分析 COBOL/old Java/.NET 程式、摘要 business logic、生成 pseudo-code、產生 unit test、建議 modern coding pattern、輔助 code translation。使用時要驗證語意一致性，因為 legacy code 可能包含隱藏 business rule、exception path、資料格式假設。GenAI output 必須經 domain expert 與 automated test 驗證。

## Summary

Legacy modernization 應先 assessment，再選擇 encapsulation、rehosting、replatforming、refactoring、rearchitecting、redesigning 或 replacing。雲端能提供強大 modernization value，但也會放大 dependency、data、security 與 operation 問題。成功關鍵是 incremental approach、POC、documentation、support runbook 與 business-aligned prioritization。
