---

---

# DevOps and Solution Architecture Framework

> Source: `aws/references/solution-architect-handbook/devops.pdf`

本章說明 DevOps、DevSecOps、CI/CD、Infrastructure as Code、configuration management、deployment strategy、continuous testing 與 CI/CD tools。核心觀念：DevOps 是 culture + automation + ownership，不只是工具鏈。

## DevOps

DevOps 促進 development 與 operations 協作，縮短 software delivery cycle，同時提升品質與穩定性。

效益：

- Faster delivery
- Better collaboration
- Higher deployment frequency
- Faster feedback loop
- Reduced deployment risk
- Improved operational ownership
- Better customer responsiveness

成熟 DevOps 要讓 team 對 service lifecycle 負責，從 build、test、deploy 到 operate 都可觀測、可自動化。

DevOps 解決的是 development 與 operations 之間目標不一致的問題。Development 追求快速交付，operations 追求穩定；若兩者分離，常導致 handoff、ticket queue、production knowledge gap。DevOps 要讓 product/service team 對 production outcome 共同負責，並用 automation 與 feedback loop 降低變更風險。

DevOps 不是導入 Jenkins 或 Kubernetes 就完成。若 deployment 仍需要大量人工核准、環境仍手動建立、production 問題仍只有 ops team 處理，那只是工具更新，不是 operating model 改變。

## CI/CD

CI (Continuous Integration) 強調頻繁合併 code，透過 automated build、unit test、static analysis、integration test 早期發現問題。

CD 可指 Continuous Delivery 或 Continuous Deployment：

- Continuous Delivery：每次變更都可被部署，但 production release 可能需人工 approval
- Continuous Deployment：通過 pipeline 後自動部署 production

CI/CD pipeline 應包含：

- Source control
- Build
- Unit test
- Static code analysis
- Security scan
- Integration test
- Artifact repository
- Infrastructure provisioning
- Deployment
- Smoke test
- Monitoring and rollback

Pipeline 也應保存 artifact immutability。Build 一次產生 artifact，後續在 test/staging/prod 推同一份 artifact，不要每個環境重新 build，否則無法保證測試過的版本就是上線版本。Artifact metadata 應包含 commit SHA、build number、dependency version、image digest、scan result。

Pipeline design 要支援 fast feedback：unit/static scan 在前面快速失敗，integration/performance/security/compliance 測試依成本與時間分層執行。Production deployment 前後要有 smoke test、health check、metric gate 與 rollback trigger。

## Continuous Monitoring and Improvement

DevOps 需要 monitoring feedback。常見 metrics：

- Deployment frequency
- Lead time for changes
- Change failure rate
- Mean time to recovery (MTTR)
- Application latency/error rate
- Infrastructure utilization
- Business metrics

Monitoring 不只看 infrastructure，也要看 application、logs、security、customer experience。

DORA metrics 是衡量 DevOps 成熟度的常見方式：deployment frequency、lead time for changes、change failure rate、MTTR。這些指標比「有多少 pipeline」更能反映 delivery capability。也要搭配 business metrics，例如 order success rate、payment failure rate、signup conversion、search latency，避免只看到 infrastructure healthy 但使用者流程失敗。

## Infrastructure as Code

IaC 讓 infrastructure provisioning 像 application code 一樣可 version、review、test、repeat。

工具：

- AWS CloudFormation
- AWS CDK
- Terraform
- Ansible
- Azure Resource Manager
- Google Cloud Deployment Manager
- Chef / Puppet

IaC 的價值：

- Standardization
- Consistency
- Reproducibility
- Compliance as code
- Faster environment creation
- Lower configuration drift

IaC 應遵循與 application code 類似的工程紀律：code review、module reuse、version pinning、state management、policy checks、drift detection、environment promotion。AWS CloudFormation/CDK 偏 AWS-native，Terraform 適合 multi-cloud 或 cross-provider resources。選型要看 team skill、state governance、module ecosystem、approval process 與 compliance requirement。

IaC 也要處理 data safety。建立/更新 resource 容易，刪除 database、bucket、KMS key、network route 可能造成重大事故。Production IaC pipeline 應有 plan review、destructive change guard、backup policy 與 explicit approval。

## Configuration Management

Configuration Management 使用 automation 標準化 resource configuration，確保 dev、test、production 環境一致。

可管理：

- OS packages
- Runtime configuration
- Application settings
- Registry/database config
- Security baseline
- Patch state

CM 降低 manual configuration error，也提升 operation productivity。

Configuration management 與 IaC 的差異：IaC 側重 provision infrastructure resources，CM 側重 resource 內部狀態，例如 OS package、agent、runtime、application config、security baseline。兩者常搭配使用：IaC 建 EC2/VPC/RDS，CM 或 image pipeline 設定 instance 內容。

Immutable infrastructure 可減少長期 configuration drift：不在既有 server 上手動修改，而是產生新 image/container，部署新版本後淘汰舊資源。若仍採 mutable server，就需要定期 drift detection 與 patch compliance。

## DevSecOps

DevSecOps 將 security controls 嵌入 DevOps lifecycle。Security 不應只在上線前檢查，而要在 code、build、test、deploy、operate 每個階段自動化。

可整合：

- SAST
- DAST
- SCA / dependency scanning
- Container image scanning
- IaC scanning
- Secret scanning
- Compliance validation
- Threat detection
- Incident response automation

目標是 early detection、continuous compliance、faster remediation。

DevSecOps 要把 security policy 變成 pipeline gate 與 runtime monitoring。例：commit 階段做 secret scanning；build 階段做 dependency/SCA 與 container image scanning；IaC 階段檢查 public S3、open security group、unencrypted database；test 階段做 DAST；deploy 後用 CloudTrail/GuardDuty/Security Hub 監控異常。

Compliance 也應自動化，例如 PCI-DSS workload 需要檢查 encryption、network segmentation、access logging、least privilege、vulnerability management。越早發現 non-compliant resource，修復成本越低。

## CD Deployment Strategies
- In-place deployment: Update application on a current server
- Rolling deployment: Gradually roll out the new version in the existing fleet of servers
- Blue-green deployment: Gradually replace the existing server with the new server
- Red-black deployment: Instant cutover to the new server from the existing server
- Immutable deployment: Stand up a new set of servers altogether

選 deployment strategy 要看 downtime tolerance、rollback requirement、state migration、database compatibility、traffic control、cost 與 user impact。任何 strategy 都必須處理 database schema migration，特別是 rolling/blue-green 下新舊版本可能同時存在，schema change 需要 backward/forward compatibility。
### In-Place Deployment

直接在 existing environment 更新 application version。簡單但風險較高，rollback 可能較困難。

In-place deployment 適合低風險、非 critical 或資源有限的系統。風險是 deployment 失敗時 production environment 已被修改，rollback 可能要重新部署舊 artifact 或 restore backup。若 server 本身也被改壞，修復時間會拉長。

### Rolling Deployment

分批更新 server fleet。優點是可達 zero downtime；缺點是新舊版本會短暫共存，需要 backward compatibility。

Rolling deployment 的 batch size、health check、pause time、rollback criteria 很重要。若每批更新太大，failure blast radius 會擴大；若 health check 太淺，可能只確認 process alive，沒有確認 business flow 正常。

### Blue-Green Deployment

Blue 是現有 production，green 是新版本完整環境。測試通過後切 traffic。優點是 rollback 快；缺點是需要額外環境成本。

Blue-green 適合需要快速 rollback 的 workload。切換可透過 load balancer、DNS、API Gateway stage、service mesh route 或 feature flag。要注意 long-lived connection、session state、database migration、background jobs 與 queue consumers 是否會同時在兩邊運作。

### Red-Black Deployment / Dark Launch

類似 blue-green，但更強調新環境先不對所有使用者開放，可用於暗中驗證功能或逐步導流。

Dark launch 可先部署功能但不對使用者開啟，或只對內部/少量 cohort 開啟。它常搭配 feature flag、A/B testing、shadow traffic。重點是 measurement：要明確比較 conversion、latency、error rate、resource usage，否則只是換一種上線方式。

### Immutable Deployment

不更新舊 instance，而是用新 image 建新 instance，再替換舊資源。適合避免 configuration drift，與 Auto Scaling、golden image、container image 搭配良好。

Immutable deployment 要求 application artifact 與 infrastructure image 可重建。若 production hotfix 透過 SSH 手動改 server，就破壞 immutable model。最佳做法是所有變更回到 source control 與 pipeline。

## Continuous Testing

CI/CD 中應在不同階段加入測試：

- Unit test
- Integration test
- Static code analysis
- Code coverage
- Performance test
- Security test
- Smoke test
- Regression test

A/B testing 與 canary analysis 可在 production 環境以少量 traffic 驗證新版本真實效果。

Continuous testing 應包含多層級：unit tests 快速驗證 function/class；integration tests 驗證 service 與 database/API；contract tests 保護 service interface；regression tests 防止舊功能破壞；performance tests 驗證 latency/throughput；security tests 驗證 vulnerability/compliance；UAT 驗證 business acceptance。

A/B testing 不只是 UI 實驗，也可以測試不同 recommendation algorithm、checkout flow、search ranking。必須定義 sample size、success metric、guardrail metric 與 rollback condition，避免錯誤實驗影響大量使用者。

## CI/CD Tooling

常見工具與服務類型：

- Code editor / IDE：AWS Cloud9、local IDE
- Source code management：GitHub、GitLab、Bitbucket、CodeCommit
- CI server：Jenkins、GitHub Actions、GitLab CI、AWS CodeBuild
- Artifact repository：container registry、package repository
- Deployment service：AWS CodeDeploy、Argo CD、Spinnaker
- Monitoring：CloudWatch、Prometheus、Grafana、Datadog

AWS DevOps toolchain 可由 CodeCommit/GitHub 觸發 CodePipeline，CodeBuild build/test artifact，ECR 儲存 container image，CodeDeploy 部署到 EC2/ECS/Lambda，CloudFormation/CDK/Terraform 建立 infrastructure，CloudWatch/CloudTrail/X-Ray 做 monitoring/audit/tracing。Jenkins 仍常見於 enterprise，但 managed CI/CD 可降低 server maintenance burden。

工具選擇要看 integration、governance、scalability、plugin ecosystem、security controls、auditability 與 team familiarity。不要為了工具流行而導入；pipeline 的可維護性比工具品牌更重要。

## Summary

DevOps 的關鍵是用 automation 建立 fast feedback loop。CI/CD、IaC、configuration management、DevSecOps、deployment strategy、continuous testing 與 monitoring 必須整合成可重複、可觀測、可回滾的 delivery system。架構師要在 deployment risk、cost、user impact 與 operation maturity 間做取捨。
