---

---

# Cloud-Native Architecture Design Patterns

> Source: `aws/references/solution-architect-handbook/cloud-native.pdf`

本章整理 cloud-native application 的主要 design patterns。重點是利用 managed services、serverless、microservices、event-driven design 與 asynchronous processing，建立 scalable、resilient、efficient 的系統。

## Cloud-Native Architecture

Cloud-native application 使用 cloud services、automation、elasticity 與 distributed architecture。它通常具備：

- Microservices 或 modular services
- Serverless 或 managed compute
- Stateless application design
- Event-driven 或 queue-based integration
- Automated scaling 與 self-healing
- Observability and monitoring
- CI/CD 與 Infrastructure as Code

Cloud-native 的風險是 provider lock-in 與 distributed system complexity，因此設計時要明確判斷 portability、operation maturity 與 team skill。

Cloud-native 不等於單純把 VM 放到 cloud 上。它的重點是 application 直接使用 cloud platform 的能力，例如 managed database、event streaming、serverless compute、managed identity、observability、autoscaling 與 global distribution。架構師要把「infrastructure management」盡量轉成「service integration and governance」，讓 team 把時間投入在 business capability，而不是 patching、capacity planning、hardware refresh。

實務設計時要先確認幾個前提：

- Workload 是否可以被拆成 independent capability，而不是把原本 monolith 用 microservice 名義重新包裝。
- Data ownership 是否清楚；每個 service 的 data model、schema evolution、backup、retention 都要有 owner。
- Team 是否具備 DevOps / CloudOps 能力，否則 distributed system 會把 operation burden 放大。
- SLA/SLO、latency、RPO/RTO、compliance requirement 是否明確，因為它們會影響 serverless、container、queue、database 與 region strategy。
- Cost model 是否可被觀測；cloud-native 的 pay-per-use 很有效，但 event storm、unbounded concurrency、excessive logging 也會快速增加成本。

## Serverless Architecture

Serverless 不是沒有 server，而是 developer 不直接管理 server。典型服務包含 AWS Lambda、API Gateway、managed database、event bus、queue、object storage。

優點：

- 不需管理 server fleet
- 自動 scale
- Pay-per-use，idle cost 低
- 適合 event-driven workload
- 可快速開發 API、batch job、notification、data processing

注意事項：

- Cold start latency
- Function timeout 與 execution limit
- Stateless design requirement
- Observability/debugging 較分散
- Vendor-specific service integration
- Concurrency limit 與 downstream pressure

典型 secure serverless architecture 會包含 API Gateway 作為入口、Lambda 執行 business logic、DynamoDB 或 Aurora Serverless 保存資料、S3 保存物件、EventBridge/SNS/SQS 做 event routing 與 decoupling、CloudWatch/X-Ray 做 logs/metrics/tracing，並透過 IAM role 套用 least privilege。若前端是 web/mobile，常搭配 Cognito 做 user authentication，CloudFront 做 global edge delivery。

設計 serverless 時不能只看 compute cost，要同時檢查：

- Invocation pattern：同步 API、非同步 event、batch、stream processing 的 timeout 與 retry behavior 不同。
- Idempotency：Lambda 可能因 retry 或 duplicate event 重複執行，寫入資料前要有 request ID、deduplication key 或 conditional write。
- Backpressure：Lambda 可以快速 scale，但 database、third-party API、legacy system 未必能承受，通常要用 SQS buffer、reserved concurrency、rate limit 或 circuit breaker。
- State externalization：function 內不要保存 session state；狀態要放在 DynamoDB、ElastiCache、S3、Step Functions execution state 等外部服務。
- Observability：每個 function 都要輸出 correlation ID、structured logs、business metrics 與 failure reason，否則跨 event chain debug 會很困難。
- Security boundary：每個 function 使用專屬 IAM role，避免一個 function 擁有整個 application 的權限。

## Stateless and Stateful Design

Stateful architecture 將 session state 保存在 server，常透過 sticky session 維持 user affinity，但 horizontal scaling 與 failure recovery 較困難。

Stateless architecture 將 session state 移到 shared store，例如 DynamoDB、Redis、RDS 或 external session service。Application server 可被任意替換，適合 Auto Scaling、blue-green deployment 與 immutable infrastructure。

設計方向：

- User session 不綁 server
- Static content 放 S3/CloudFront
- Application server 只處理 request/business logic
- State 放 database/cache/object store

Stateful design 不是錯誤，只是需要被明確管理。Gaming session、real-time collaboration、long-running workflow、stream processing consumer、database connection pool 都可能需要 state。若必須 stateful，要明確設計：

- Session affinity 失效時的 fallback，例如 reconnect、session restore、state replication。
- Instance replacement 時資料是否會遺失；local disk、in-memory queue、temporary file 都不能被視為 durable state。
- Scale-in protection 與 graceful shutdown，避免 Auto Scaling 移除 instance 時中斷使用者工作。
- Backup、snapshot、replication 與 failover 流程。

Stateless design 的主要優勢是讓 compute tier disposable。當 application server 不保存 user context 時，load balancer 可以把 request 發到任意 healthy instance；blue-green deployment 可以直接切 traffic；failed instance 可以直接丟棄重建；capacity 可以跟 demand 變動。

## Microservice Architecture

Microservices 將大型系統拆成 bounded contexts。每個 service 管理自己的 domain logic 與 data，透過 API/event 溝通。

設計原則：

- Service boundary 依 domain 而不是 technical layer 切分
- 每個 service 可獨立 deploy、scale、monitor
- 避免 shared database 造成 coupling
- 使用 API contract 或 event schema 管理整合
- 建立 distributed tracing、logging、metrics

好處是降低 change risk、提升 scalability 與 fault isolation；代價是 monitoring、governance、data consistency、service coordination 更複雜。

Microservice architecture 的切分應從 business domain 開始，例如 catalog、order、payment、inventory、shipment、recommendation，而不是 web layer、service layer、DAO layer 這種 technical tier。每個 service 應該能有自己的 release cadence、owning team、data store、scaling policy 與 operational dashboard。

需要避免的設計問題：

- Shared database：多個 service 直接讀寫同一套 schema，會讓 schema change 變成全系統協調。
- Excessive synchronous calls：一個 user request 串接太多 service，latency 與 failure probability 會疊加。
- No contract governance：API/event schema 沒版本管理，consumer 會被 producer change 破壞。
- Missing distributed tracing：如果 request 跨 10 個 service，沒有 trace ID 幾乎無法定位 latency 與 failure。
- One pipeline for all services：若所有 service 必須一起 build/deploy，就不是有效的 microservice。

AWS 上常見組合包含 ECS/EKS/Fargate 承載 container services、API Gateway 或 ALB 作入口、Cloud Map 做 service discovery、App Mesh 做 service-to-service traffic management、DynamoDB/Aurora/OpenSearch 等服務各自持有資料、EventBridge/SNS/SQS/Kinesis 做 asynchronous integration。

## Saga Pattern

Saga pattern 用來處理 distributed transaction。跨多個 service 的 business operation 被拆成一連串 local transactions；每一步成功後發 event 給下一步，失敗時執行 compensating transaction。

適用：

- Order/payment/shipping 等跨 service 流程
- Long-running transaction
- Eventual consistency 可接受的場景

注意：

- Compensation logic 必須清楚
- Event ordering、duplicate event、retry 要處理
- Business team 需接受 eventual consistency model

Saga 有兩種常見協調方式：

- Choreography：每個 service 發出 event，下一個 service 自行訂閱並執行。優點是 decoupled，缺點是流程分散，難以看出整體 transaction state。
- Orchestration：由 central orchestrator 管理流程，例如 AWS Step Functions。優點是流程清楚、容易觀測與補償，缺點是 orchestrator 會成為流程控制中心。

以 e-commerce order 為例：Order service 建立訂單、Payment service 授權付款、Inventory service 保留庫存、Shipping service 建立配送。若 Shipping 失敗，可能要釋放庫存並取消付款授權。這些 compensating transactions 必須是 business-valid，不是單純 database rollback。

## Fan-Out/Fan-In Pattern

Fan-out/fan-in 將一個大型工作拆成多個 parallel tasks 處理，再聚合結果。適合 analytics、image processing、large-scale event processing。

優點：

- 提升 throughput
- 降低單一 task duration
- 適合 serverless parallelization

風險：

- Coordination complexity
- Partial failure handling
- Aggregation consistency
- Resource spike 與成本控制

Fan-out/fan-in 的常見 AWS 實作是：一個 request 觸發 Step Functions Map state 或 Lambda fan-out，將工作分派到多個 Lambda/ECS task/SQS messages，最後把結果存入 DynamoDB/S3 或由 aggregator Lambda 合併。適合 image resizing、video transcoding、多資料來源 enrichment、parallel API calls、large report generation。

設計重點是 partial failure。單一子任務失敗時，要決定整個工作失敗、重試該 shard、回傳 partial result，或把 failed item 放入 DLQ。若工作量很大，也要限制 parallelism，避免 downstream database 或 third-party API 被瞬間打爆。

## Service Mesh Pattern

Service mesh 在 microservices 之間提供 traffic management、service discovery、mTLS、retry、circuit breaking、observability。它把 cross-cutting concerns 從 application code 移到 infrastructure layer。

適合 service 數量多、需要一致 network/security policy 的系統。缺點是增加 platform complexity，需要成熟 operation 能力。

Service mesh 通常透過 sidecar proxy 或 data plane 管理 service-to-service traffic，control plane 則負責 policy distribution。它能提供：

- Fine-grained routing：canary、weighted routing、traffic shadowing。
- Resilience policy：timeout、retry、circuit breaker、outlier detection。
- Security policy：mTLS、service identity、authorization policy。
- Observability：per-service latency、error rate、request volume、trace metadata。

AWS App Mesh 是 AWS 的 managed service mesh 選項，可搭配 ECS、EKS、Fargate、EC2 workloads。導入前要確認是否真的需要 service mesh；如果 service 數量少，ALB/API Gateway + application-level client library 可能更簡單。

## Reactive and Queue-Based Architecture

Reactive architecture 強調 asynchronous、non-blocking、event-driven。Queue-based architecture 讓 producer 與 consumer decouple，避免上游等待下游。

常見 pattern：

- Queue chain：多階段 sequential workflow
- Job observer：監控 long-running job 狀態
- Pipes-and-Filters：將 data processing 拆成可組合 filter
- Publisher/Subscriber：一個 event 發送給多個 subscriber
- Event stream：處理 continuous events，例如 clickstream、IoT telemetry

Reactive architecture 適合需要即時反應、event volume 大、consumer 數量多或處理時間不穩定的 workload。核心是讓 producer 不等待所有 consumer 完成，因此系統整體能承受 downstream slowness。

Queue chain pattern 把處理分成多段 queue 與 worker，例如 image upload -> virus scan -> resize -> metadata extraction -> publish。每一段可獨立 scale，失敗時也只卡在特定 queue。

Job observer pattern 適合 long-running job，例如報表產生、ML training、資料匯入。Client 送出 job 後取得 job ID，observer 或 status API 回報 queued/running/succeeded/failed，避免 HTTP request 長時間等待。

Pipes-and-Filters architecture 把資料轉換流程拆成多個 filter，每個 filter 做單一任務，例如 validate、normalize、enrich、aggregate、store。這讓 pipeline 比較容易測試與替換。

Publisher/subscriber model 適合一個 event 需要觸發多個 business reaction，例如 order created 同時通知 payment、inventory、analytics、email。AWS 上可用 SNS、EventBridge 或 Kafka topic。

Event stream model 適合 continuous data，例如 clickstream、ad tracking、IoT telemetry、application logs。AWS 上常用 Kinesis Data Streams、Amazon MSK、Kinesis Data Firehose、Lambda、S3、Athena、Redshift 或 OpenSearch，分別處理 ingestion、processing、storage 與 analytics。

## BFF Pattern

BFF (Backend for Frontend) 為不同 frontend 建立專屬 backend，例如 web、mobile、partner API 各自有最佳化 API。優點是 UX/performance 更好，避免一個 generic API 滿足所有 client。代價是 backend 數量增加，需要治理 duplicated logic。

BFF 對 mobile 特別有價值，因為 mobile network latency、payload size、battery usage 都更敏感。Mobile BFF 可以把多個 backend calls 聚合成一個 optimized response，也可以用不同 DTO 避免 client 處理過多資料。Web BFF 則可服務 browser-specific rendering、cookie/session、A/B testing 或 personalization。

治理原則是：BFF 可以做 aggregation、composition、client-specific shaping，但不要把 core domain logic 複製到每個 BFF。Domain rule 仍應放在 backend domain services。

## Anti-Patterns

常見 cloud-native anti-pattern：

- Distributed monolith：表面是 microservices，實際 deployment/data/release 仍高度耦合
- Chatty services：service 間過度同步呼叫造成 latency
- Shared database across services：破壞 service autonomy
- 忽略 observability：failure 後無法定位
- Serverless function 過大或責任不清

PDF 也強調幾個常見但容易被忽略的 anti-pattern：

- Single point of failure：缺少 redundancy、failover、multi-AZ 或 queue buffer。
- Manual scaling：靠人工擴容，無法應對 traffic spike。
- Tightly coupled services：一個 service change 需要多個 team 同步 release。
- Ignoring security best practices：沒有 least privilege、encryption、secret management、network segmentation。
- Not monitoring or logging：沒有 metrics、logs、tracing、alerts，系統故障時只能猜。
- Ignoring network latency：跨 region/service/database 的 round trip 沒有被設計進 latency budget。
- Lack of testing：沒有 unit/integration/load/failure testing，cloud-native complexity 會在 production 爆發。
- Over-optimization：過早導入 service mesh、多層 cache 或複雜 eventing，增加不必要維運成本。
- Not considering costs：serverless、stream、logging、cross-AZ/cross-region transfer 都可能成為隱性成本。

## Summary

Cloud-native patterns 的共同目的，是降低 coupling、提升 elasticity、改善 fault isolation，並讓系統能快速演進。Serverless、stateless design、microservices、Saga、fan-out/fan-in、service mesh、queue、event stream、BFF 都有明確適用情境；架構師要同時評估 team maturity、operation complexity、consistency requirement 與成本。
