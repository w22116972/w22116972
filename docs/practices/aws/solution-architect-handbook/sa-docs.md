---

---

# Solution Architecture Document

> Source: `aws/references/solution-architect-handbook/sa-docs.pdf`

本章說明 Solution Architecture Document (SAD) 的目的、views、structure、lifecycle、best practices、pitfalls 與 RFx procurement documents。核心觀念：SAD 是讓 business 與 technical stakeholders 對 solution design 形成共同理解與正式 agreement 的文件。

## Purpose of SAD

SAD 提供 end-to-end application view，避免團隊在需求、設計、技術選型、operation、security、integration 上各自理解。

SAD 的目標：

- Provide high-level architecture overview
- Capture different architecture views
- Align business and technical stakeholders
- Support implementation and maintenance
- Document decisions and trade-offs
- Provide modernization/assessment baseline
- Support procurement and vendor communication

SAD 的價值在於降低 ambiguity。Business stakeholder 用它確認 solution 是否解決正確問題；developer 用它理解 component、interface、data model；operations 用它建立 monitoring/runbook；security 用它檢查 trust boundary；procurement/vendor 用它對齊 scope 與 deliverables。

SAD 也可作為 assessment baseline。未來 modernization、incident review、audit 或 vendor transition 時，SAD 能快速說明系統如何組成、為什麼這樣設計、還有哪些 open risks。

## Views of SAD

SAD 應包含適合不同 stakeholder 的 views：

For non-technical users：

- Business view
- Process view
- Logical view

For technical users：

- Application view
- Development view
- Deployment view
- Operational view
- Data view
- Security view

可使用 UML、C4 model、deployment diagram、ERD、sequence diagram 等表示。

不同 view 不應重複堆疊同一張大圖。Business/process view 應說明 value stream 與流程；logical/conceptual view 說明主要 capabilities；application view 說明 modules/service；development view 說明 code/package/repository；deployment view 說明 runtime topology；operational view 說明 monitoring、support、backup、DR；data/security view 則聚焦資料與風險控制。

## SAD Structure

### Solution Overview

用簡潔語言說明 solution purpose、scope、major components、business value。需讓 business user 能理解。

Solution overview 可用 e-commerce order/supply chain workflow 這類業務語言描述：customer 下單、系統檢查庫存、處理付款、建立出貨、通知 customer、更新 order status。這一節不要塞滿服務名稱，應先讓 stakeholder 理解 end-to-end flow。

### Business Context

說明 business problem、stakeholders、goals、constraints、assumptions、success metrics。

Business context 應列出 in-scope/out-of-scope、business drivers、current pain points、target outcomes、critical constraints、regulatory requirement、timeline、budget、success metrics。若假設未被確認，應放入 assumptions/open issues，避免後續被誤認成已決定事實。

### Conceptual Solution Overview

提供抽象層級圖，呈現主要 modules 與 information flow，是 business 與 technical stakeholder 的共同語言。

Conceptual view 要保持在「可讓非技術 stakeholder 理解」的層次。例如顯示 customer portal、order management、payment provider、inventory system、warehouse/shipping、notification、analytics，而不是直接畫所有 subnet/security group。它是後續詳細 architecture 的入口。

### Solution Architecture

深入說明各 architecture domains。

#### Information Architecture

包含 website navigation、taxonomy、wireframe、高層資訊流。

Information architecture 對 UX 與 content-heavy system 特別重要。它應說明使用者如何瀏覽資訊、主要 user journey、page/screen relationship、navigation hierarchy、metadata/taxonomy。這能讓 UX designer 產生 wireframe，也讓 development team 理解資訊結構。

#### Application Architecture

列出 application components/modules、responsibilities、interfaces、dependencies，以及哪些 module 需要 retire、retain、replatform、transform。

Application architecture 應描述每個 component 的責任、owner、interface、runtime、scaling pattern、dependency、deployment unit。若是 modernization project，應標示哪些 module 會 retire、retain、rehost、replatform、refactor。也要記錄 synchronous/asynchronous calls、API contracts、batch jobs 與 error handling。

#### Data Architecture

說明 data objects、database、ERD、data flow、data lifecycle、data ownership、sensitive data。

Data architecture 應包含 entity relationship、cardinality、data store choice、data flow、data retention、classification、encryption、backup、migration、archiving、lineage。ERD 不只是 database admin 文件；它讓 business 與 engineering 理解資料關係，例如 customer、order、payment、shipment、product、inventory 之間如何關聯。

#### Integration Architecture

列出 upstream/downstream systems、API、message queues、file transfer、protocol、data exchange format。

Integration architecture 應具體列出 integration endpoint、protocol、payload/schema、authentication、frequency、latency expectation、retry、error handling、idempotency、owner。對 e-commerce，可能包含 payment gateway、warehouse management、CRM、email/SMS、tax service、analytics pipeline。Integration 是 production failure 高風險區，不能只畫箭頭。

#### Infrastructure Architecture

說明 deployment environment、network、compute、storage、load balancing、availability zones、DR。

Infrastructure architecture 應分 environment：dev、QA、UAT、staging、production。內容包含 account/subscription、region/AZ、VPC/subnet、routing、load balancer、compute、container/serverless、database/storage、DNS、certificate、backup、DR、monitoring。圖應能讓 operations/security team 判斷 traffic path 與 trust boundary。

#### Security Architecture

包含 authentication、authorization、network boundary、encryption、audit、compliance、threat model。

Security architecture 應明確說明 identity provider、SSO/MFA、IAM roles、network segmentation、WAF/firewall、secrets management、KMS/encryption、logging/audit、data classification、compliance requirement、threat model summary。若有 customer/vendor access，要說明 trust model 與 access review。

### Solution Implementation

說明 delivery approach、deployment plan、migration strategy、testing、cutover、rollback、dependency。

Solution implementation 需要把 architecture 轉成可執行計畫：milestones、team responsibility、environment readiness、data migration, deployment strategy、test strategy、release gating、cutover window、rollback plan、training、communication。若是 cloud migration，要標示 migration waves 與 validation criteria。

### Solution Management

聚焦 production support、monitoring、alerting、runbook、incident response、maintenance ownership。

Solution management 應涵蓋 monitoring dashboard、alert severity、on-call/escalation、SLA/SLO、backup/restore、patching、capacity management、cost monitoring、security operation、DR test、known issues、support handoff。這一節常被忽略，但決定系統能否長期運作。

### Appendix

放 references、decision log、glossary、detailed diagrams、assumptions、open issues。

Appendix 適合放不影響主線閱讀但需要保存的資料，例如 vendor comparison、detailed network ports、API schema links、compliance mapping、capacity calculation、meeting notes、ADR、glossary。主文件應保持可讀，細節放 appendix。

## SAD Lifecycle

SAD 是 running document，不是一次性 deliverable。它應在 project early stage 建立，隨著 design、implementation、operation 持續更新。

Lifecycle：

- Initial draft
- Stakeholder review
- Architecture refinement
- Formal approval
- Implementation updates
- Operational updates
- Modernization updates

SAD lifecycle 通常從 early design 建立 initial draft，經 stakeholder review 修正，再取得 formal approval。Implementation 期間應更新實際設計差異；production 後應由 operations/change process 觸發更新。若 SAD 不更新，它很快會變成錯誤資訊來源，比沒有文件更危險。

每次重大 architecture decision 都應記錄 decision rationale：選了什麼、拒絕了什麼、原因、風險、假設、後續 review 條件。這能避免幾個月後團隊忘記 trade-off。

## Best Practices and Pitfalls

Best practices：

- 針對不同 audience 提供合適 detail
- 圖與文字一致
- 記錄 assumptions、constraints、trade-offs
- 保持文件可維護
- 與 code、IaC、runbook、API spec 對齊
- 定期 review/update

Pitfalls：

- 只有 technical details，business user 看不懂
- 文件過時
- 沒有記錄 decision rationale
- diagrams 與 reality 不一致
- 缺少 operation/security/integration view

好的 SAD 應「足夠詳細但不變成資料垃圾場」。過度簡化會無法指導 implementation；過度堆疊會沒人讀。建議用 executive summary + domain sections + appendix 分層。圖表要標明 scope、legend、data flow direction、trust boundary，且與文字保持一致。

常見 pitfall 還包括：只描述 happy path、沒有 failure/rollback、沒有 owner、沒有 data classification、沒有 vendor dependency、沒有 cost/operation impact、沒有把 open questions 明確列出。

## IT Procurement Documentation

Solutions Architect 也常參與 RFx：

- RFI (Request for Information)：蒐集市場/廠商資訊
- RFQ (Request for Quotation)：取得價格報價
- RFP (Request for Proposal)：要求 vendor 提供完整 solution proposal

RFP 最常見，因為 IT solution 通常技術複雜、長期影響大。架構師需協助定義 technical requirement、evaluation criteria、integration/security/compliance needs。

RFx 差異：

- RFI：用來了解市場、vendor capability、可行方案，通常不要求詳細價格與承諾。
- RFQ：需求已明確，主要比較價格、交付時間、商業條款。
- RFP：問題與目標明確，但 solution approach 需要 vendor 提案，會比較技術、方法、風險、成本、support、timeline。

Solutions Architect 在 procurement 中應協助撰寫 technical scope、non-functional requirements、architecture constraints、security/compliance controls、integration requirements、evaluation matrix、POC criteria、vendor Q&A。對 vendor proposal 的評估不應只看價格，也要看 architecture fit、operability、lock-in、scalability、security、support model。

## Summary

SAD 的價值是 alignment 與 maintainability。它應同時服務 business、technical、operation、security、procurement stakeholders，並以不同 architecture views 呈現 solution。好的 SAD 不是文件形式完整而已，而是能支援決策、實作、維運與未來演進。
