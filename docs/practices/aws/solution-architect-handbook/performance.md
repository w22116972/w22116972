---

---

# Performance Considerations

> Source: `aws/references/solution-architect-handbook/performance.pdf`

本章整理 performance optimization 的設計原則與技術選型。核心觀念：performance 是跨 compute、storage、database、network、application、mobile、testing、monitoring 的 continuous improvement。

## Design Principles for High Performance

Performance efficiency 是用合適資源滿足 workload requirement。設計時要先定義目標：

- Latency
- Throughput
- Concurrency
- Availability under load
- User experience
- Cost-performance ratio

Performance design 要先建立 measurable target，例如 p95 latency < 300 ms、checkout throughput 1000 TPS、batch job 2 小時內完成、mobile first paint < 2 秒。沒有 target 的優化很容易變成主觀調參。

Performance efficiency 也包含 cost-performance。用更大的 instance 可以暫時解決問題，但不一定是最有效方案。架構師要比較 vertical scaling、horizontal scaling、cache、database/index tuning、async processing、CDN、data partitioning、algorithm improvement。

## Reducing Latency

Latency 是 request 到 response 的時間，會受多層影響：

- Network distance
- Router hops
- Transmission medium
- DNS lookup
- Server processing
- Application dependency
- Database read/write
- Storage I/O
- Cache miss

低 latency 通常也能提升 throughput，但需要測量每一層 bottleneck。

Latency 要拆成 client-side、network、edge、application、dependency、database/storage。常見改善方式：

- 使用 CDN/edge cache 讓 static content 靠近使用者。
- 使用 Route 53 latency-based routing 或 global accelerator 降低跨區延遲。
- 減少 synchronous dependency calls，改用 async/queue 或 aggregation。
- 使用 connection reuse、HTTP keep-alive、compression、payload trimming。
- 避免 chatty API，mobile/web client 一次 request 取得必要資料。
- 用 cache/read replica 降低 database read latency。
- 對高 latency dependency 設 timeout、retry budget、circuit breaker。

## Improving Throughput

Throughput 是單位時間可處理的資料量或 request 數量。可出現在：

- Network throughput
- Disk throughput
- Database throughput
- Application request throughput
- Stream processing throughput

設計方法：

- 選擇合適 instance type
- 增加 parallel processing
- 使用 cache
- 使用 load balancer
- 對 read-heavy workload 使用 read replica
- 適當 partition/sharding
- 優化 storage IOPS

Throughput bottleneck 可能在不同層：network bandwidth、load balancer target、application worker、thread pool、database connection pool、disk IOPS、queue consumer 或 external API quota。只看 CPU 容易誤判。

提升 throughput 的常見架構做法是水平擴展 stateless compute、將 read/write 分離、把同步工作轉成 queue-based processing、使用 batch/bulk API、增加 partition 並行處理、針對 hot key/hot partition 做資料分散。對 streaming workload，要同時調整 shard/partition 數、consumer parallelism、checkpoint frequency。

## Handling Concurrency and Parallelism

Concurrency 是同時處理多個工作；parallelism 是真正同時執行多個工作。兩者都會影響 application design。

注意事項：

- Thread/process pool
- Connection pool
- Lock contention
- Database transaction isolation
- Queue depth
- Backpressure
- Rate limiting

Database concurrency 特別複雜，因為 read/write 同時操作會涉及 consistency、locking、transaction。

Concurrency 問題常表現為 thread starvation、connection exhaustion、lock wait、deadlock、queue backlog、rate limit exceed。Application 應明確設定 worker pool、connection pool、request timeout 與 retry policy，避免 unlimited concurrency 壓垮 downstream。

Parallelism 適合可拆分工作，例如 image processing、ETL partition、ML batch inference、report generation。需要注意 task coordination、partial failure、result aggregation 與 cost spike。

## Applying Caching

Caching 可降低 latency 與 backend load。

層級：

- CPU cache
- OS page cache
- Application memory cache
- Database cache
- DNS cache
- CDN cache
- Redis/Memcached

需設計 TTL、eviction、cache invalidation、stale data policy。

Cache strategy 要按資料特性選擇。Static assets 適合長 TTL + filename versioning；product catalog 可接受短暫 stale；inventory/price/payment 狀態通常需要更嚴格 consistency。Cache miss、cache stampede、hot key、eviction storm 都可能造成 production incident。

常見保護方式包含 request coalescing、jittered TTL、pre-warming、negative caching、rate limiting、multi-layer cache。要監控 cache hit ratio、eviction count、memory usage、latency、backend load。

## Compute Choices

Compute 不只等於 server。可選：

- CPU：general-purpose workload
- GPU：parallel computation、ML、graphics
- FPGA：特殊硬體加速
- ASIC：特定用途高效率計算
- VM / bare metal
- Containers
- Serverless functions

選型依 workload：monolith 可先用 VM/bare metal；microservices 適合 containers；event-based function 適合 serverless。

CPU-bound workload 需要看 clock speed、vCPU、NUMA、vector instruction；memory-bound workload 要看 memory size/bandwidth；GPU 適合 deep learning training/inference、video processing、graphics；FPGA/ASIC 適合低延遲或高度特化運算。AWS instance family 選擇應依 workload profile，而不是使用單一 instance type 跑全部系統。

Compute architecture 也要考慮 scaling speed。EC2 啟動需要時間；container task 啟動較快；Lambda 可快速併發但受 cold start、timeout、concurrency limit 影響。

## Containers

Docker 將 application 與 dependency 打包，提升 portability 與 consistency。Kubernetes 負責 orchestration、scheduling、service discovery、self-healing、scaling。

Cloud providers 也提供 ECS、EKS、GKE、AKS 等 managed container platforms。Containers 對 multi-cloud/hybrid portability 有幫助，但仍需治理 image security、resource limit、observability。

Container performance 要設定 CPU/memory requests 與 limits。Requests 太低會被排到資源不足節點，limits 太低會 throttling 或 OOMKill。Kubernetes/ECS scaling policy 要根據 CPU、memory、request rate、queue depth 或 custom metrics 設計。Container image 要精簡，避免大 image 拖慢 rollout 與 cold start。

## Serverless

Serverless 讓 function independent scaling，適合 event-driven workload、API backend、scheduled jobs、stream processing。

優點：

- Pay-per-use
- 自動 scale
- 無 server management

限制：

- Cold start
- Timeout
- Concurrency limit
- Vendor-specific integration

Serverless performance 需要關注 cold start、package size、runtime、VPC networking、memory setting、provisioned concurrency、downstream capacity。Lambda memory 調高也會增加 CPU allocation，某些 workload 反而能以更短 duration 降低總成本。對 stream/event source，要調整 batch size、parallelization factor、retry/DLQ。

## Storage Choices

### Block Storage / SAN

Block storage 將資料切成 block，適合 database、low-latency random I/O、VM disk。例如 AWS EBS。

### File Storage / NAS

File storage 以 directory/file 方式提供 shared access，適合 shared filesystem、content repository、lift-and-shift workload。例如 EFS、FSx。

### Object Storage

Object storage 以 object + metadata + API 存取，scale 高、成本低，適合 static content、backup、data lake、media。例如 Amazon S3。

Multi-cloud/hybrid storage 需注意 data locality、latency、data sovereignty、replication cost。

Storage performance 指標不同：block storage 看 IOPS、throughput、latency；file storage 看 metadata operation、concurrent access、locking；object storage 看 request rate、prefix/partition、multipart upload、eventual consistency behavior、data transfer。選錯 storage type 會讓 application 即使 compute 很大也跑不快。

S3 適合大規模 object 與 static content，不適合需要 POSIX file locking 的 application。EFS/FSx 適合 shared filesystem。EBS 適合單 instance attached block storage 與 database disk。資料離 compute 太遠會產生 network latency 與 egress cost。

## Database Choices

### OLTP / Relational Database

適合 transactional workload，需要 ACID、complex transaction、structured schema。Performance optimization 包含 index、query tuning、read replica、partition、connection pooling。

Relational database 常見 bottleneck 是 slow query、缺 index、過度 index、lock contention、connection storm、transaction 太長。Performance tuning 要用 query plan、slow query log、wait event、cache hit ratio、IOPS、replication lag 判斷，不要只升級 instance。

### NoSQL

適合 semi-structured/unstructured、高 scale、高 write/read throughput、flexible schema。需依 access pattern 選 key-value、document、column-family、graph。

NoSQL performance 通常取決於 partition key/access pattern。Hot partition 會讓整體 throughput 被單一 key 限制。設計前要列出 query patterns，包含 primary key lookup、range query、secondary index、sort/filter、TTL、item size。DynamoDB 類服務尤其需要在 table/index design 階段決定 access pattern。

### OLAP / Data Warehouse

適合 analytics/reporting。MPP 與 columnar storage 可提升 scan/aggregation performance，例如 Redshift、Netezza、SQL Server data warehouse。

Data warehouse performance 取決於 data distribution、sort key、partition、compression、materialized view、query concurrency、workload management。報表查詢不應直接打 transactional database；應建立 analytical model 或 data mart。

### Search

大量 log、text、event search 適合 OpenSearch/Elasticsearch 類型服務。

Search performance 取決於 index design、shard count、replica count、mapping、refresh interval、query pattern、heap/memory。Search store 適合作為 read/search optimized copy，不應替代 transactional source of truth。

## Network and Global Users

全球使用者要降低 network latency：

- CDN
- Edge location
- Route 53 latency-based routing
- Global accelerator
- Regional deployment
- Load balancer
- Auto Scaling

Global user performance 需要同時處理 static content、dynamic API 與 data locality。Static content 可用 CloudFront；dynamic API 可用 regional endpoint、Global Accelerator 或 latency routing；資料若必須跨 region 讀寫，要評估 replication lag、consistency 與 compliance。Load balancer 應搭配 health check、connection draining、TLS policy 與 target scaling。

## Mobile Performance

Mobile app 需考慮慢網路、舊設備、電量、payload size、offline mode、image/video loading、API round trips。可透過 compression、pagination、local cache、progressive loading 改善。

Mobile performance 最常見問題是 API round trip 太多與 payload 太大。BFF pattern、GraphQL、response shaping、pagination、image resizing、adaptive bitrate、offline cache、background sync 都能改善體驗。API 要支援 retry/idempotency，因為 mobile network interruption 很常見。

## Testing and Monitoring

Performance testing 必須包含：

- Load test
- Stress test
- Spike test
- Soak test
- Concurrency test
- Mobile/network condition test

Monitoring 要追蹤 latency、throughput、error rate、saturation、queue depth、DB metrics、cache hit ratio。

Performance test 要模擬真實 workload，而不只是固定 TPS。要包含 ramp-up、peak、spike、soak、failure during load、dependency slowness、mobile network condition。結果應產生 baseline，後續 release 才能比較是否 regression。

Monitoring 應使用 RED/USE 指標：Request rate、Error rate、Duration；Utilization、Saturation、Errors。對 distributed system 要有 tracing，才能知道 latency 花在 API、service、database、cache 或 third-party dependency。

## Summary

High performance 不是單一優化，而是從 latency、throughput、concurrency、cache、compute、storage、database、network 與 client experience 做整體設計。架構師應先定義 performance target，再用測試與 monitoring 驗證，避免只靠直覺選更大的 server。
