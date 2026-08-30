---

---

# Data Engineering for Solution Architecture

> Source: `aws/references/solution-architect-handbook/data-engineering.pdf`

本章整理 big data architecture、data pipeline、ingestion、storage、processing、analytics 與常見 data architecture patterns。核心觀念：data architecture 要從 data characteristics、latency requirement、access pattern、governance 與 business question 反推。

## Big Data Pipeline

Big data pipeline 的一般流程：

1. Data source
2. Data ingestion
3. Data storage
4. Data processing / transformation
5. Analytics / visualization
6. Insight and decision

架構設計要關注 time to answer，也就是 data 產生到能回答 business question 的 latency。不同需求會導向 batch、streaming 或 hybrid pipeline。

Analytics team 常見障礙包含 data silo、資料品質不穩、schema 不一致、資料來源太多、pipeline tightly coupled、無法追蹤 lineage、缺少 metadata catalog、資料存取權限流程過慢。Solution architect 的工作不是只挑工具，而是設計一條能讓資料被可靠收集、治理、處理、查詢與重用的 flow。

Big data pipeline 設計要先回答：

- Business question 是什麼，dashboard、ad hoc query、ML feature、real-time alert 或 regulatory report？
- Latency requirement 是 minutes、seconds、sub-second 還是 daily batch？
- Data volume/velocity/variety 有多大？
- 是否需要 replay raw data？
- 是否需要保留原始資料以便 audit 或重新處理？
- 資料誰擁有，誰可以存取，如何分類與遮罩？

## Pipeline Decoupling

常見錯誤是把 ingestion、storage、processing、analytics 綁成一條 tightly coupled pipeline。建議 decouple：

- Ingestion layer：收資料
- Storage layer：保留 raw/curated data
- Processing layer：清理、轉換、聚合
- Analytics layer：查詢、dashboard、ML feature

Decoupling 可降低重跑成本、支援多種 consumer，並讓 pipeline 更容易擴展。

Decoupling 的實作通常是先將 raw data landing 到 durable storage，再由多個 processing jobs 產生 curated datasets。這樣 ingestion failure 不會直接破壞 analytics layer，processing logic 變更也可以從 raw data 重跑。若 ingestion 直接寫入 reporting schema，任何 schema change 或 transformation bug 都會污染下游。

典型分層：

- Raw zone：保存原始資料，不做或只做最小轉換。
- Cleansed zone：做格式標準化、去重、資料品質檢查。
- Curated zone：依 business domain 建立可查詢 dataset。
- Consumption zone：供 BI、ML、API、data sharing 使用。

## Data Ingestion

Data ingestion 包含 batch ingestion 與 streaming ingestion。

Streaming data 例如 clickstream、IoT telemetry、logs、financial transactions。通常先進入 Kafka、Kinesis、Pub/Sub 等 streaming storage，再由 processing system 消費。

常見 ingestion tools：

- Apache Kafka
- Apache Flume
- Apache Sqoop
- Apache NiFi
- Apache Storm / Samza
- AWS Kinesis
- AWS Glue
- Azure Event Hubs
- Google Pub/Sub

選型時看 throughput、latency、ordering、durability、retry、schema evolution 與 operation complexity。

File-based ingestion 適合 partner feed、IoT batch dump、legacy export、log archive。大量檔案進入 object storage 時，要處理 small files problem、partitioning、file format、schema drift 與 arrival notification。

Database ingestion 常用 CDC 或 scheduled extract，把 operational database 變更送到 lake/warehouse。需注意 source database 負載、transaction ordering、delete/update semantics、PII masking 與 schema evolution。

Streaming ingestion 適合 clickstream、payment event、sensor telemetry、ad tracking、log analytics。設計時要決定 partition key、retention period、consumer group、retry/DLQ、late event handling 與 exactly-once/at-least-once tradeoff。

Cloud ingestion 在 AWS 常見服務包含 Kinesis Data Streams、Kinesis Data Firehose、AWS Glue、Database Migration Service (DMS)、S3 event notification、API Gateway/Lambda、Amazon MSK。選擇 managed service 可降低 operation burden，但要確認 throughput limit、pricing model 與 integration pattern。

## Data Storage

選擇 data store 取決於：

- Data structure：structured、semi-structured、unstructured
- Access pattern：read-heavy、write-heavy、search、analytics
- Latency requirement
- Consistency requirement
- Volume and velocity
- Cost
- Governance/security

Data storage 設計應分開考慮 system of record 與 analytical copy。Operational system 的 database 不應被大量 BI query 直接壓垮；通常會透過 ETL/ELT 或 CDC 把資料送到 warehouse/lake。資料生命週期也要定義 hot/warm/cold/archive tier，避免所有資料都放在高成本儲存。

資料格式會影響成本與效能。CSV/JSON 易讀但查詢效率差；Parquet/ORC 這類 columnar format 適合 analytics，可搭配 partitioning 降低掃描量。Compression、partition key、file size、metadata catalog 都是 data lake performance 的關鍵。

## Structured Data Stores

Relational database 適合 transactional system，提供 ACID consistency 與複雜 query。常見於 banking、order、payment、inventory。

Data warehouse 是整合多來源資料的 central repository，適合 reporting、BI、analytics。Columnar storage 與 MPP 可提升 analytical query performance，例如 Amazon Redshift。

Relational database 以 normalized schema、transaction integrity、SQL query 為主，適合 OLTP。Data warehouse 以 denormalized/star schema、columnar storage、MPP query 為主，適合 OLAP。兩者的 workload pattern 不同，不應把 transaction database 當成企業報表平台。

Data warehouse 的限制是資料常被物理集中與複製，容易產生資料新鮮度、同步、成本與 ownership 問題。Modern architecture 常把 warehouse 與 data lake 搭配，讓 raw data 留在 lake，curated/aggregated data 進 warehouse。

## NoSQL Databases

NoSQL 適合 high-scale、semi-structured 或 flexible schema 場景。

類型：

- Key-value store：DynamoDB、Redis
- Document store：MongoDB
- Column-family store：Cassandra、HBase
- Graph database：關聯與 graph traversal

SQL vs NoSQL 主要取捨在 consistency、schema flexibility、query pattern、horizontal scalability。

NoSQL 選型應由 access pattern 驅動。Key-value store 適合低延遲查詢與 predictable key access；document store 適合 JSON-like entity；wide-column store 適合大規模 write/read 與 sparse columns；graph database 適合多跳關係，例如 fraud ring、recommendation、social graph。

NoSQL 不是 SQL 的替代品，而是針對特定 scale/access pattern 的設計。若 application 需要複雜 joins、ad hoc query、強 transaction consistency，relational database 仍可能更合適。

## Search, Object, Vector, Blockchain, Streaming Stores

- Search data store：OpenSearch/Elasticsearch，適合 log search、clickstream search、full-text search
- Object storage：S3 類型服務，scale 高、成本低、flat namespace，適合 data lake、raw files、media
- VectorDB：儲存 embedding，支援 similarity search、recommendation、RAG
- Blockchain data store：提供 immutability、distributed ledger，適合 supply chain、finance audit 等場景
- Streaming data store：支援 continuous data ingestion 與 replay，例如 Kafka/Kinesis

Search store 通常為查詢最佳化副本，不應作為唯一 system of record。Object storage 是 data lake 的基礎，因為它可保存不同格式、不同 schema、不同生命週期的資料。Vector database 在 GenAI/RAG 架構中保存 document embeddings，查詢時用 similarity search 找到相關 context。Blockchain data store 適合多方需要不可竄改紀錄的場景，但成本、throughput 與 governance 要審慎評估。

Streaming store 與 queue 不完全相同。Queue 通常工作被 consumer 消費後移除；stream 允許多個 consumer group 依 retention period replay event。這對 real-time analytics、audit replay、ML feature generation 很重要。

## Data Processing and Analytics

Processing 目標是將 raw data 轉成可分析、可查詢、可視覺化的資料。

常見處理：

- Cleaning
- Deduplication
- Enrichment
- Aggregation
- Format conversion
- Feature engineering
- Batch ETL / ELT
- Stream processing

工具包含 Spark、Flink、Glue、EMR、Dataflow、Databricks、Redshift、Athena 等。

Batch processing 通常用於每日報表、歷史資料重算、大規模 transformation。Stream processing 用於即時 alert、fraud detection、online metrics、IoT monitoring。Hybrid pattern 會將 streaming data 寫入 lake，同時產生 real-time aggregate，後續再用 batch job 做精確校正。

AWS reference flow 常見是：raw files 進 S3，Glue crawler/catalog 建 metadata，Glue/EMR/Spark 做 ETL，curated data 放 S3 或 Redshift，Athena 直接查 S3，QuickSight 做 dashboard。若資料是 streaming，Kinesis/MSK 進來後可由 Lambda/Flink/Spark Streaming 處理，再進 S3/OpenSearch/Redshift。

Processing layer 要處理 data quality：schema validation、null/outlier check、duplicate detection、referential consistency、late arriving data、quarantine bad records。沒有 quality gate 的 data pipeline 會讓 BI 與 ML output 失去可信度。

## Data Architecture Patterns

### Data Lake

Data lake 以 object storage 儲存 raw、semi-structured、unstructured data，成本低且可 scale。需要 metadata catalog、schema governance、data quality，否則容易變成 data swamp。

Data lake 通常包含 ingestion、raw zone、processed zone、catalog、query engine、security layer。AWS 上可用 S3 作 storage、Glue Data Catalog 管 metadata、Lake Formation 管權限、Athena/EMR/Redshift Spectrum 查詢。設計重點是 partitioning、file format、access control、encryption、lifecycle policy 與 lineage。

### Lakehouse

Lakehouse 結合 data lake 的低成本與 data warehouse 的 governance/query 能力，支援 transaction、schema enforcement、BI 與 ML workload。

Lakehouse 透過 table format 和 metadata layer 改善 data lake 的一致性與治理，例如 schema evolution、ACID-like transaction、time travel、upsert/delete。適合希望同一份資料同時支援 BI、data science 與 ML feature engineering 的組織。

### Data Mesh

Data mesh 將 data ownership 分散到 domain teams，data 作為 product 提供。需要 common governance、data catalog、quality standard 與 self-service platform。

Data mesh 適用於大型組織中 centralized data team 成為 bottleneck 的情境。每個 domain team 交付 data product，包含 schema、quality SLA、documentation、access policy、lineage 與 support channel。Central platform team 提供共通工具，確保 security/governance 不因 decentralization 失控。

### Streaming Architecture

Streaming architecture 適合 real-time dashboard、fraud detection、IoT、clickstream analysis。需處理 ordering、late arrival、windowing、backpressure、exactly-once 或 at-least-once semantics。

Streaming pipeline 要特別注意 time semantics：event time、processing time、watermark、window size 都會影響結果。例如 ad tracking 或 clickstream analytics 可能需要同時做 real-time dashboard 與 batch correction。資料可先進 Kinesis/MSK，再分流到 Lambda/Flink 做即時處理、Firehose/S3 做長期保存、OpenSearch 做 search dashboard、Redshift 做分析。

## Best Practices

- 從 business question 反推 data pipeline
- Raw data 與 curated data 分層保存
- Pipeline decouple，避免單一路徑綁死
- 建立 data catalog 與 lineage
- 定義 data quality checks
- 根據 hot/warm/cold data 選擇 storage tier
- 對 sensitive data 做 classification、encryption、access control
- 監控 ingestion lag、processing latency、failure rate、cost

另外要避免幾個常見問題：

- 把所有資料放進單一 database，導致 OLTP、analytics、search、ML 互相干擾。
- 沒有 raw data retention，transformation bug 後無法重跑。
- 缺少 metadata catalog，使用者不知道資料定義與可信度。
- 忽略 small files 與 partition design，造成 query cost 高且速度慢。
- 沒有 data lineage，無法追蹤 dashboard 指標來源。
- 權限設計只靠 bucket/table level，無法處理 PII、row-level、column-level 控制。

## Summary

Data engineering architecture 要把 data ingestion、storage、processing、analytics 拆清楚，並根據 data type、volume、velocity、latency 與 access pattern 選擇技術。Data lake、lakehouse、data mesh、streaming architecture 都不是萬用答案；重點是讓資料可治理、可重用、可查詢，並能支援 BI、analytics、ML 與 business decision。
