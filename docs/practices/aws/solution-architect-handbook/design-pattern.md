---

---

# Solution Architecture Design Patterns

> Source: `aws/references/solution-architect-handbook/design-pattern.pdf`

本章整理常見 solution architecture patterns。重點是理解每個 pattern 解決的問題、適用場景與 trade-off，而不是把 pattern 當固定模板套用。

## N-Tier Layered Architecture

N-tier architecture 將系統拆成多個 layer，常見是 three-tier：

- Web layer：user-facing，處理 UI、static/dynamic web content、UX、page load
- Application layer：business logic、workflow、API、algorithm
- Database layer：transactional data、user profile、order、session、application state

優點是 separation of concerns，方便各 layer 獨立 scale 與修改。缺點是 layer 間仍可能形成同步依賴，需要 cache、load balancer、queue 降低 coupling。

Three-tier 在 e-commerce、blog、content site 很常見。Web layer 接受 user request 並處理 presentation；application layer 執行 catalog browsing、order creation、comment posting 等 business logic；database layer 保存 product、order、user、payment state。每一層可以有不同 scaling policy，例如 web layer 用 CDN/cache，application layer 用 Auto Scaling，database layer 用 read replica、partitioning 或 managed HA。

Layering 的風險是所有 request 都必須穿越固定層級，若 application layer 或 database layer 成為 bottleneck，整體 performance 仍會受限。架構師要用 caching、asynchronous processing、read model、queue 或 domain split 降低單一路徑壓力。

## Multi-Tenant SaaS Architecture

SaaS 常採 multi-tenant architecture：多個 tenant 共用 application/platform，但資料與設定需隔離。

設計重點：

- Tenant isolation
- Authentication and authorization
- Per-tenant configuration
- Shared database vs isolated database
- Billing and metering
- Noisy neighbor control
- Compliance and data residency

Multi-tenant 可降低 provider 成本、簡化維運，但 isolation、customization、security 必須設計清楚。

Tenant isolation 可用不同層級實作：

- Shared application + shared database + tenant ID：成本最低，但 row-level isolation 必須非常嚴格。
- Shared application + separate schema/table：隔離較好，維運複雜度增加。
- Shared application + separate database：資料隔離與客製化更強，成本更高。
- Separate stack per tenant：適合高價值或高度合規 tenant，但 provisioning、upgrade、monitoring 成本最高。

Multi-tenant SaaS 還需要設計 per-tenant metering、usage throttling、quota、billing、custom domain、feature flag、data export/delete、audit log。Noisy neighbor 是核心風險：單一 tenant 的大量 traffic 不應拖垮其他 tenant。

## Service-Oriented Architecture

SOA 將功能拆成 services，透過 protocol 溝通。它比 monolith 更容易重用與整合，但需要 service governance。

傳統 SOA 常使用 SOAP；現代架構更常使用 RESTful API、event-driven 或 microservices。

SOA 的重點是把 reusable business capability 以 service 形式暴露，例如 customer profile、payment authorization、inventory check、shipping quote。它可以降低重複實作，但若 governance 過重、service catalog 不清、ESB 成為 central bottleneck，就會拖慢 delivery。

SOA 與 microservices 的差異不只是技術。Microservices 通常強調 independent deployment、team ownership、bounded context 與 decentralized data management；SOA 在 enterprise integration 場景常更強調 service reuse、標準化與集中治理。

## RESTful Web Service Architecture

RESTful architecture 使用 HTTP method 與 resource-oriented URL 建立 service interface。

常見 HTTP method：

- GET：讀取 resource
- POST：建立 resource 或觸發 action
- PUT/PATCH：更新 resource
- DELETE：刪除 resource

設計重點：

- Resource naming 清楚
- Stateless request
- Appropriate status codes
- JSON/XML payload
- Versioning
- Authentication/authorization
- Rate limiting
- Error contract

RESTful API 設計要注意 resource granularity。太粗會讓 client 拿到過多資料；太細會造成 chatty API 與高 latency。Pagination、filtering、sorting、partial response、idempotency key、correlation ID、request validation、consistent error format 都是 production API 必要細節。

E-commerce REST example 可包含 `/products`、`/products/{id}`、`/carts/{id}`、`/orders`、`/orders/{id}`。GET 應 safe/idempotent，PUT 應 idempotent，POST 通常建立新 resource。對 payment/order creation 這類操作，idempotency key 可避免 client retry 造成重複扣款或重複訂單。

## Cache-Based Architecture

Caching 將 data/content 暫存到更接近使用者或 application 的位置，以降低 latency 與 backend load。

常見 cache layer：

- Browser cache
- DNS cache
- CDN edge cache
- Proxy cache
- Application cache
- Database cache

### Cache Distribution Pattern

在 three-tier architecture 中，static content 可透過 CDN cache，application frequent queries 可透過 Redis/Memcached，database 可使用 internal cache 或 read replica。

### Rename Distribution Pattern

使用 CDN 時，更新 static file 可改 filename/version，例如 `app.v2.js`，避免等待舊 cache expiration。比大規模 invalidation 成本更低。

### Cache Proxy / Rewrite Proxy Pattern

Proxy 放在 user 與 backend 之間，可 cache content、改寫 destination 或協助 gradual migration。需避免 proxy 成為 single point of failure。

### App Caching Pattern

Application 與 database 之間加入 cache engine。

- Lazy caching：read-heavy、可接受 stale data
- Write-through：資料更新時同步寫 cache，降低 stale risk

Redis vs Memcached 取決於 data structure、persistence、replication、cluster、operation model。

Cache 的核心難題是 consistency。TTL 太短會降低 cache hit ratio；TTL 太長會造成 stale data。Write-through 可降低 stale risk，但 write latency 較高；lazy loading 簡單且適合 read-heavy，但第一次讀取會 cache miss；write-around 可避免一次性資料污染 cache；write-back 可提高效能但資料遺失風險更高。

Memcached 通常簡單、低延遲、適合 ephemeral key-value cache。Redis 支援更豐富 data structures、persistence、replication、pub/sub、sorted set，適合 session store、leaderboard、rate limiter、distributed lock 等場景。選 Redis 也要處理 memory eviction、persistence setting、cluster slot、failover 與 hot key。

CDN/cache distribution pattern 常用於 static assets、product images、downloadable content。Rename distribution pattern 透過 filename versioning 避免全站 invalidation。Proxy/rewrite proxy 可支援 migration 或 content routing，但必須 HA 部署並監控 cache hit ratio、origin latency、error rate。

## MVC and DDD

MVC (Model-View-Controller) 將 presentation、business/data model、control logic 拆開，提升 maintainability。

DDD (Domain-Driven Design) 強調從 domain model、bounded context 與 ubiquitous language 建立系統邊界，適合複雜 business domain 與 microservices。

MVC 的價值是讓 view、controller、model 各自演進。Online bookstore example 中，View 顯示 catalog/order UI，Controller 接受 search/add-to-cart/checkout request，Model 保存 book、customer、order、inventory 等 domain state。若 View 直接讀寫 database 或 Controller 塞滿 business rules，maintainability 會快速下降。

DDD 則要求 architecture 語言與 business domain 對齊。Bounded context 可避免不同 domain 對同一名詞有不同意義卻混用，例如「customer」在 billing、support、marketing 中可能代表不同資料與行為。DDD 也常搭配 aggregate、entity、value object、repository、domain service、domain event，協助拆分 microservice 邊界。

## Resilience Patterns

### Circuit Breaker

Circuit breaker 偵測 downstream failure，暫停請求，避免等待與 cascading failure。適合 remote service call、unstable dependency、microservices。

Circuit breaker 通常有 closed/open/half-open 狀態。Closed 時正常通過；錯誤率或 timeout 超過 threshold 進入 open，快速失敗或回 fallback；一段時間後進入 half-open，以少量 request 測試 downstream 是否恢復。這能避免 thread pool 被慢速 dependency 塞滿。

### Bulkhead

Bulkhead 將 resource pool 隔離，例如 thread pool、connection pool、service instance pool，避免一個功能耗盡資源拖垮整體。

Bulkhead 的概念來自船艙隔離。實作上可用獨立 queue、獨立 worker pool、獨立 database connection pool、獨立 Auto Scaling group 或獨立 service。例：standard user request 與 admin/reporting request 使用不同資源池，避免報表查詢拖垮 checkout。

### Floating IP

Floating IP 可在 failover 時從 primary node 切換到 standby node，讓 client 使用同一個 IP 存取服務。適合 HA 場景，但在 cloud-native 架構中常被 load balancer/DNS failover 替代。

Floating IP 適合傳統 active/passive deployment，例如 primary instance 故障時把 virtual IP 移到 standby instance。Cloud 上常用 Elastic IP reassociation、route table update、load balancer target health 或 DNS failover 達成類似效果。設計時要確認 failover time、ARP/DNS/cache 行為與 split-brain 防護。

## Containers

Containers 將 application 與 dependencies 打包，提升 portability、deployment consistency 與 resource utilization。適合 microservices、CI/CD、immutable deployment。

需搭配 orchestration，例如 Kubernetes、ECS，處理 scheduling、self-healing、service discovery、scaling。

Containers 的核心優勢是 package once, run consistently。Developer 將 runtime、library、application code 打包成 image，CI pipeline 掃描並推到 registry，orchestrator 在 cluster 中排程 container。這能降低「在我機器上可跑」的問題。

Container deployment 要設計 image versioning、base image patching、secret injection、resource requests/limits、health probes、rolling update、service discovery、persistent storage、network policy 與 log collection。Container 本身不解決 stateful data 問題，database/state 仍要外部化或使用專門 stateful orchestration。

## Database Handling Patterns

Database architecture 需考慮：

- Primary/standby failover
- Read replica
- Backup/restore
- Sharding/partitioning
- Cache
- Connection pooling
- Data consistency
- Transaction boundary

Application architecture 不應把 database 當無限可 scale 的單點資源。

Database scaling pattern 包含 vertical scaling、read replica、sharding、partitioning、caching、CQRS/read model、archive old data。Read replica 適合 read-heavy workload，但 write bottleneck 仍在 primary。Sharding 可水平擴展寫入，但會增加 query routing、cross-shard transaction、rebalancing 與 operational complexity。

High-availability database pattern 通常包含 primary/standby、replication、automatic failover、backup/snapshot、point-in-time recovery。Application 要能處理 failover 後 connection reset、DNS endpoint change、temporary write unavailability。

## Clean Architecture and Anti-Patterns

Clean Architecture 強調 business rules 不依賴 framework、UI、database 或 external systems，讓核心 domain 更可測試、可維護。

常見 anti-pattern：

- Big ball of mud
- God service
- Shared database everywhere
- Tightly coupled services
- Cache without invalidation strategy
- No observability
- Over-engineering without business need

Clean Architecture 的核心是 dependency rule：business rules 不依賴 UI、database、framework 或 external service。外層 adapter 負責 web、database、messaging、third-party integration；內層 domain 保持可測試與穩定。這能讓 framework 替換、database migration、API 變更時不直接污染 business logic。

Anti-pattern 需要用 evidence 判斷。不是所有 monolith 都壞，也不是所有 microservices 都好。如果系統規模小、team 小、domain 還不穩，過早拆成 distributed services 可能只是增加 deployment 與 debugging 成本。

## Summary

Architecture pattern 是解決特定問題的設計語言。N-tier、SaaS multi-tenancy、SOA、REST、cache patterns、MVC、DDD、circuit breaker、bulkhead、containers、database HA、Clean Architecture 都應依 business requirement、team maturity、operation cost 與 failure mode 選擇。
