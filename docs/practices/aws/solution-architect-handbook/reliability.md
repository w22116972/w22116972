---

---

# Architectural Reliability Considerations

> Source: `aws/references/solution-architect-handbook/reliability.pdf`

本章整理 reliability、high availability、self-healing、distributed system、RTO/RPO、replication 與 disaster recovery strategy。核心觀念：reliability 是系統在故障中維持服務與恢復的能力，必須透過設計、測試與 operation 驗證。

## Design Principles

### Self-Healing by Automation

可靠系統應能偵測 failure 並自動修復：

- Health checks
- Auto Scaling replacement
- Restart failed containers
- Automated failover
- Automated backup/restore validation
- Alarm-triggered remediation

Monitoring 要追蹤 KPI/SLO，並根據 threshold 或 anomaly 觸發 response。

Self-healing 的設計目標不是「永遠不壞」，而是把常見 failure mode 自動化處理，降低 MTTR。典型做法包含：

- Load balancer health check 發現 unhealthy target 後停止送流量。
- Auto Scaling group 或 container orchestrator 自動替換 failed instance/task。
- Database 使用 Multi-AZ、read replica 或 managed failover。
- Queue consumer 失敗時以 retry policy、visibility timeout、DLQ 隔離毒性訊息。
- CloudWatch alarm / EventBridge rule 觸發 Systems Manager Automation 或 Lambda remediation。
- Backup job 完成後自動做 restore test，而不是只檢查 backup file 存在。

自動修復要有 guardrail。錯誤的 auto-remediation 可能擴大事故，例如誤判健康檢查導致全部 instance 被輪流重建。因此要保留 alarm、rate limit、manual override 與 clear audit trail。

### Quality Assurance

QA 環境要能重現 development/production configuration，避免因環境不一致造成 defect。IaC 與 configuration management 可提升 reproducibility。

Reliability 的 QA 不只測 functional correctness，也要測 failure behavior。應在接近 production 的環境中驗證：

- Deployment rollback 是否可用。
- Dependency unavailable 時 application 是否 degrade gracefully。
- Database failover 後 connection pool 是否能恢復。
- Backup restore 後資料是否完整且符合 RPO。
- Scaling event 是否會造成 session loss 或 queue backlog。
- IAM、security group、routing、DNS 變更是否會破壞 recovery path。

### Distributed System

Monolith 容易因單點 failure 影響整體。Distributed architecture 可將 workload spread across multiple resources，降低 blast radius。

常見 mechanism：

- Load balancing
- Circuit breaker
- Bulkhead
- Retry with backoff
- Queue-based decoupling
- Multi-AZ deployment

Distributed system 的可靠性來自 failure isolation。Load balancer 避免單一 instance 故障；bulkhead 把不同 workload 隔離，避免 reporting job 拖垮 checkout；circuit breaker 在 downstream failure 時快速失敗並保護資源；queue 則讓 producer 與 consumer 解耦。

但 distributed system 也引入新問題：network partition、partial failure、event duplication、clock skew、eventual consistency、configuration drift。架構師必須在 design review 中明確討論這些問題，而不是只畫出更多元件。

### Monitoring and Capacity

傳統 on-premises 需要預估 peak capacity；cloud 可透過 Auto Scaling 依 demand 增減資源。Capacity planning 仍需建立 baseline 與測試 scaling behavior。

Capacity management 要同時看 resource metrics 與 business metrics。例如 CPU 低不代表 checkout 正常，可能是 queue 卡住或 downstream payment 失敗。常見指標包含 request rate、p95/p99 latency、error rate、queue depth、database connections、replication lag、cache hit ratio、thread pool saturation、disk IOPS 與 business conversion rate。

Scaling policy 要測試 scale-out speed 與 scale-in side effect。過慢的 scale-out 會讓 spike 期間 error rate 上升；過快的 scale-in 可能中斷 long-running job。對 stateful workload，要加 termination drain、connection draining 或 scale-in protection。

### Recovery Validation

Backup 不等於 recoverable。必須定期做 recovery validation，確認 RTO/RPO 可達成，runbook 可執行。

Recovery validation 應該包含 tabletop exercise、automated restore test、game day、region/AZ failover drill 與 post-test review。每次演練都要記錄實際 RTO/RPO、手動步驟、失敗 dependency、DNS propagation time、data reconciliation step 與 communication flow。若演練結果不符合 SLA，就要回到 architecture backlog 處理，而不是只更新文件。

## RPO and RTO

- RTO (Recovery Time Objective)：系統可接受停機多久
- RPO (Recovery Point Objective)：可接受遺失多少資料

RTO/RPO 越低，cost、complexity、redundancy 越高。應依 business criticality 決定。

兩者常被混淆：

- 如果 RTO 是 4 小時，代表 service 從災難到恢復最多可停 4 小時。
- 如果 RPO 是 15 分鐘，代表最多可接受遺失 15 分鐘資料。

對不同系統應分級：public website 可能要求很低 RTO 但資料量少；payment/order 系統通常要求低 RTO 與低 RPO；internal reporting 可接受較長恢復時間。不要對所有 workload 套同一個 DR 等級，否則成本會失控。

## Data Replication

### Synchronous Replication

資料寫入 primary 與 replica 都完成才回應。資料落差低，RPO 接近 0，但 latency 較高，受距離影響。

### Asynchronous Replication

Primary 先完成寫入，replica 稍後同步。Latency 低、成本較低，但可能有 replication lag 與資料遺失風險。

### Replication Methods

- Array-based replication
- Network-based replication
- Host-based replication
- Hypervisor-based replication
- Database-native replication

每種方法在 cost、RPO、RTO、platform dependency、operation complexity 上不同。

資料複製設計需要區分：

- Snapshot / backup：適合 point-in-time recovery，但 restore time 較長。
- Database-native replication：例如 read replica、logical replication、binlog replication，通常較理解 transaction semantics。
- Storage-level replication：對 application 透明，但不一定理解 database consistency。
- Application-level replication/event sourcing：彈性高，但需要更多 application logic。

同步複製提供較佳 RPO，但跨距離 latency 會直接影響 write path；非同步複製效能較好，但 failover 時可能要處理 replication lag 與資料衝突。跨 region DR 時通常要特別評估資料主權、latency、cost 與 compliance。

## Disaster Recovery Strategies

DR strategy 依 RTO/RPO 由高到低可分：

### Backup and Restore

成本最低，RTO/RPO 最高。保存 machine image、database snapshot、backup 到 DR site，災難時還原。

適合非 critical workload 或可接受長時間恢復的系統。

Backup and restore 的關鍵不是「有沒有備份」，而是備份是否可被完整還原。要確認 backup encryption key 可用、IAM 權限可用、restore target capacity 可建立、network/security settings 可重建、application configuration 與 secret 也能恢復。若只備份資料、不備份 IaC 與 configuration，實際 RTO 會被環境重建拖長。

### Pilot Light

在 DR site 保留核心資料層或 critical services，以小規模持續運作。災難時啟動 application/server 並 scale up。

比 backup and restore 成本高，但 RTO/RPO 較低。

Pilot light 通常保留最關鍵且最難重建的資料層，例如 database replication、minimal directory/identity、core network、minimal monitoring。Application server、web tier 或 worker 在 disaster 發生時才用 IaC/AMI/container image 快速啟動。它適合「資料不能丟太多，但可接受一段啟動時間」的 workload。

### Warm Standby

DR site 有完整但低 capacity 的 application/database stack，持續同步 primary site。災難時擴大 capacity 並切 traffic。

適合 critical workload，但成本與 operation complexity 更高。

Warm standby 的 DR environment 通常已具備完整 stack，只是 capacity 比 primary 小。切換時要擴容 compute、調整 Auto Scaling、提升 database capacity、切 DNS/traffic routing，並確認所有 external integrations 指向正確 endpoint。它比 pilot light 更快，但也要求兩邊環境長期保持一致。

### Multi-Site

Primary 與 DR site 都完整運作並服務 traffic。RTO/RPO 接近 0，但成本最高，且需要 global routing、data consistency、conflict handling、operation maturity。

Multi-site / active-active 設計需要處理最困難的資料一致性問題。若兩個 region 都可寫入同一份 business data，就必須設計 conflict resolution、idempotency、global transaction boundary 或 eventual consistency model。Global DNS/traffic management 可以使用 latency-based routing、weighted routing、health check failover，但 application 本身也必須能承受跨 region dependency failure。

## DR Best Practices

- 根據 business impact 定義 RTO/RPO
- 定期測試 failover/failback
- 保留 documented runbook
- 使用 IaC 快速重建 infrastructure
- 監控 replication lag
- 保護 backup encryption/access
- 測試 dependency recovery
- 確認 DNS/routing 切換
- 演練 communication plan

DR plan 至少應包含：

- Scope：哪些 application、data store、external dependency 在 DR 範圍內。
- Ownership：incident commander、application owner、database owner、network/security owner。
- Trigger criteria：何時宣布 disaster，避免過早/過晚切換。
- Runbook：failover、validation、communication、failback 的實際步驟。
- Validation checklist：功能、資料完整性、security、performance、business sign-off。
- Dependency map：identity provider、DNS、certificate、payment gateway、email/SMS、third-party API。
- Audit and compliance：DR test evidence、backup retention、access logs。

## Cloud Reliability

Cloud 提供 built-in reliability capabilities：

- Multi-AZ
- Auto Scaling
- Managed database failover
- Monitoring/alerting
- Backup automation
- Systems Manager patching
- Change tracking
- Infrastructure as Code

但 cloud 不會自動讓 application reliable；application architecture 仍需設計 failure handling。

Cloud reliability capability 應該對應到設計目標：

- Multi-AZ：降低 single data center failure。
- Auto Scaling：因應 demand change 與 instance failure。
- Managed database failover：降低 database operation burden。
- S3 durability / versioning / lifecycle：保護物件資料。
- CloudWatch / X-Ray / CloudTrail：提供 monitoring、tracing、audit。
- AWS Backup：集中化 backup policy。
- Route 53 health check / failover routing：支援 DNS-level failover。
- Systems Manager：patching、inventory、automation、run command。

可靠性仍然是 shared responsibility。AWS 保護 cloud infrastructure，customer 仍負責 application architecture、data backup policy、IAM、configuration、testing 與 incident process。

## Summary

Reliability 來自 self-healing automation、distributed design、capacity management、replication、DR planning 與 recovery validation。架構師必須根據 business requirement 選擇 backup and restore、pilot light、warm standby 或 multi-site，並用實際演練證明 RTO/RPO 可達成。
