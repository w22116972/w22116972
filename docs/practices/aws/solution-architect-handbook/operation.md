---

---

# Operational Excellence Considerations

> Source: `aws/references/solution-architect-handbook/operation.pdf`

本章整理 operational excellence 的設計原則與實作方法。核心觀念：operation 是持續改善系統的能力，包含 planning、monitoring、incident response、analytics、RCA、audit 與 CloudOps。

## Design Principles

### Automating Manual Tasks

IT operations 不能依賴手動流程。應自動化 provisioning、deployment、patching、backup、monitoring、alerting、remediation。

Automation 可降低 human error，讓團隊從 reactive firefighting 轉向 proactive operation。

自動化範圍應涵蓋整個 lifecycle：環境建立、config 套用、artifact deploy、database backup、certificate renewal、patching、log retention、security response、scale action、resource cleanup。手動 runbook 可以存在，但高頻與高風險步驟應逐步轉成 script、pipeline 或 managed automation。

### Incremental and Reversible Changes

Operational change 應小步、可回滾：

- Small batch changes
- Blue-green / canary / rolling deployment
- Versioned configuration
- Rollback plan
- Change validation

Incremental change 可降低 blast radius。一次只改小範圍，配合 feature flag、canary、blue-green 或 rolling deployment，可以在問題擴大前停止。Reversible change 則要求 rollback path 已測試，不只是文件上寫「回復上一版」。

Database/schema change 是最常破壞 reversible 的部分。應採 expand-and-contract pattern：先新增 backward-compatible schema，再部署支援新舊 schema 的 application，最後清理舊欄位/流程。

### Predicting Failures and Responding

透過 metrics、logs、trend analysis、ML anomaly detection 預測 failure，並根據 SLA、RTO、RPO 建立 response scenario。

Predictive operation 需要 baseline。CPU 慢慢升高、disk 成長趨勢、queue depth 累積、database connection 接近上限、certificate 即將過期、error rate 異常，都應在造成 outage 前觸發 action。CloudWatch anomaly detection、APM、log analytics、capacity forecast 都可支援。

### Learning from Mistakes

每次 incident 都應透過 RCA 形成 action item，更新 runbook/playbook，避免相同問題重複發生。

Post-incident review 應聚焦 system improvement，而不是責備個人。有效 RCA 會產出具體 action：新增 alert、修正 timeout/retry、改善 deployment gate、補齊 dashboard、移除 manual step、加強 test、更新 dependency map。

### Keeping Runbook Updated

Runbook 應記錄 routine operation、incident history、steps、validation result、owner、escalation path。它要 people-independent，避免知識只存在特定人身上。

Runbook 要包含可執行指令、預期輸出、判斷條件、rollback step、何時升級、誰能 approve。若 runbook 只寫高層流程，incident 時仍然無法縮短 MTTR。Runbook 更新應是 change/release/incident process 的一部分。

## Planning for Operational Excellence

Planning phase 需定義 operational priorities：

- SLA/SLO
- Critical workloads
- Dependency mapping
- Monitoring scope
- Incident response process
- Change management
- Compliance requirement

Operational planning 要在 architecture design 階段完成。若 production 後才補 monitoring、backup、alert、on-call、runbook，通常會漏掉 critical dependency。架構師應建立 workload criticality 分級，對每一級定義 logging retention、backup frequency、support hours、alert severity、RTO/RPO、deployment window。

## IT Asset Management

ITAM 追蹤 IT resources 與 lifecycle，協助 tactical/strategic decision。

應包含：

- Asset discovery
- Ownership
- Cost
- License
- Lifecycle state
- Dependency
- Security/compliance posture

Asset management 在 cloud 中更重要，因為 resource 建立很快也容易被遺忘。ITAM/CMDB 應與 cloud inventory 同步，追蹤 instance、database、bucket、load balancer、IP、certificate、KMS key、IAM role、license。缺少 owner 的 resource 應進入 cleanup 或 ownership review。

## Configuration Management

Configuration management 讓 operation team 掌握 resource configuration、dependency、change history。

用途：

- Reduce downtime
- Support security analysis
- Maintain compliance
- Track configuration drift
- Support incident response

Cloud 中可用 AWS Config、Systems Manager、Trusted Advisor 等服務協助。

Configuration management 要能回答「目前 production 實際狀態是什麼」以及「它和期望狀態有何差異」。AWS Config 可追蹤 resource configuration history 與 compliance；Systems Manager 可做 inventory、patch、run command、automation；Trusted Advisor 可提出 cost/security/performance/reliability 建議。

Configuration drift 是 production 事故來源之一。若工程師手動修改 security group、instance package、environment variable，IaC 與實際環境就會不一致。Drift detection 與 change control 要把這類行為納入治理。

## Monitoring System Health

Operational excellence 的 functioning phase 依賴 proactive monitoring。

### Infrastructure Monitoring

Metrics 包含：

- CPU
- Memory
- Disk usage
- Network in/out
- IOPS
- Instance/container health

Infrastructure monitoring 要配合 capacity planning。CPU、memory、disk、network、IOPS 只是低層訊號，必須與 autoscaling policy、SLO、queue backlog、deployment event 關聯。若 alarm 太多且不可行動，on-call 會產生 alert fatigue。

### Application Monitoring

Metrics 包含：

- Endpoint availability
- Latency
- Error rate
- Request count
- Dependency failure
- Business transaction success rate

Application monitoring 應追蹤 golden signals：latency、traffic、errors、saturation。還要納入 business transaction，例如 login success、checkout success、payment authorization、search zero-result rate。只有 system metrics 沒有 business metrics，會看不到 customer-impacting failure。

### Platform Monitoring

需監控 database、queue、cache、container platform、Kubernetes、serverless、managed services 的 service-specific metrics。

Platform monitoring 要理解各 managed service 的特定限制。Database 看 connections、replication lag、lock wait、storage growth；queue 看 depth、age of oldest message、DLQ；cache 看 hit ratio、evictions、memory；Lambda 看 duration、throttle、concurrency、error；Kubernetes 看 pod restart、pending pods、node pressure。

### Log Monitoring

Centralized logging 讓 team 能查詢、分析、建立 alert。工具包含 CloudWatch Logs、Logstash、Splunk、Google Cloud Logging。

Log 要結構化並包含 correlation/request ID，否則 microservices 與 event-driven workflow 很難追蹤。Logging 也要控制成本；debug log 若長期開啟，cloud log ingestion/storage 會快速增加。Retention policy、PII masking、access control、archive policy 必須明確。

### Security Monitoring

應監控：

- Unauthorized access
- Suspicious traffic
- Malware/threat detection
- Network logs
- IAM activity
- Data access

AWS GuardDuty、Security Hub、WAF logs 等可支援 cloud security monitoring。

Security monitoring 要與 incident response 串接。CloudTrail 管 API audit，VPC Flow Logs/WAF logs 看 network/web traffic，GuardDuty 偵測威脅，Security Hub 聚合 findings。關鍵是 triage workflow：誰接收 finding、如何判斷 severity、如何 containment、如何追蹤 remediation。

## Alerts and Incident Response

Alert 需要 severity 與 response process：

- Critical：立即影響 customer/business
- High：可能快速惡化
- Medium：需排程處理
- Low：觀察或 backlog

Incident response 應定期演練，確認 escalation、runbook、communication、rollback、recovery steps 可用。

Alert 設計要 action-oriented。每個 page 都應有明確 owner、runbook、impact、threshold reason。Severity 定義要和 customer impact 對齊，而不只是 metric threshold。Incident response 需要角色分工：incident commander、communications lead、technical lead、scribe。重大事故後要做 timeline、impact analysis、root cause、corrective actions。

## Improving Operational Excellence

### IT Operations Analytics

ITOA 收集多系統 logs/metrics/events 到 central storage，透過 big data pipeline 做 correlation、trend analysis、prediction。

可使用 S3/data lake、ETL、BI dashboard、ML anomaly detection。

ITOA 的價值是把 logs、metrics、events、traces、tickets、deployment records 集中分析。它可以用於容量預測、incident correlation、異常偵測、cost optimization、SLA reporting。例如部署後 error rate 上升，可將 deployment event 與 application metrics 自動關聯。

### Root Cause Analysis

RCA 目的不是找人負責，而是找出 system/process gap。Five Whys 是常見方法。結果應轉成 preventive action、runbook update、monitoring improvement。

RCA 要避免停在「工程師操作錯誤」這種表面原因。更有價值的問題是：為什麼系統允許錯誤操作？為什麼沒有 automated validation？為什麼 rollback 不可用？為什麼 monitoring 沒提前發現？這些才會導向 architecture/process improvement。

### Auditing and Reporting

Audit 用於發現 malicious activity、policy violation、idle resource、license issue、data access risk。對 compliance、security、cost optimization 都重要。

Reporting 需要服務不同 audience：executive 需要 SLA/cost/risk summary；operations 需要 incident trend、MTTR、capacity；security 需要 access/audit/compliance；engineering 需要 error、latency、deployment health。報表若不能驅動 action，就只是噪音。

## Operational Excellence in Public Cloud

AWS 可用服務：

- CloudWatch
- CloudTrail
- AWS Config
- Systems Manager
- Trusted Advisor
- GuardDuty
- Security Hub
- Cost Explorer

這些服務應對應到 operational capability：

- CloudWatch：metrics/logs/alarms/dashboard。
- CloudTrail：API audit trail。
- AWS Config：configuration history/compliance。
- Systems Manager：patch、inventory、automation、run command。
- Trusted Advisor：cost/security/performance/reliability recommendations。
- GuardDuty/Security Hub：threat detection 與 finding aggregation。
- Cost Explorer/Budgets：cost visibility 與 guardrail。

## CloudOps

CloudOps 是 cloud environment 的操作方法，涵蓋 provisioning、deployment、monitoring、security、governance、cost management、incident response。

Key pillars：

- Automation
- Monitoring
- Security and compliance
- Cost management
- Governance
- Continuous improvement

CloudOps 與傳統 operations 的差異在於 infrastructure 由 API 驅動，change rate 更高。這要求 policy-as-code、automation、tagging、central logging、account governance、cost controls、self-service platform。CloudOps team 不應成為所有 ticket 的 bottleneck，而應提供 reusable guardrails 與 platform capability。

## Summary

Operational excellence 是持續迭代的能力。架構師應在設計中納入 ITAM、configuration management、monitoring、alerting、incident response、RCA、audit、CloudOps 與 automation，讓系統不只可上線，也能長期穩定、可治理、可改善。
