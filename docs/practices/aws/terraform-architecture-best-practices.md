# Terraform 架構與最佳實務：面試深度指南

Terraform 是 declarative Infrastructure as Code (IaC) 工具。使用者描述 desired infrastructure；Terraform 讀取 configuration、state 與 provider 回傳的 remote objects，建立 dependency graph 和 execution plan，再透過 provider 呼叫 AWS APIs 使實際環境收斂。

這份指南放在 **AWS Cloud** 類別，因為 Terraform 的主要責任是 VPC、IAM、EKS、databases、DNS 等 cloud infrastructure。它不屬於 Kubernetes；Terraform pipeline 的 scanning、approval 與 credentials 雖然屬於 DevSecOps controls，但不是整份 architecture 文件的分類。

## 核心架構與執行流程

```text
Git repository / reusable modules
               |
               v
      Terraform CLI or remote runner
        | configuration + variables
        | provider schemas
        | prior state + refreshed remote state
        v
       Dependency graph -> execution plan -> apply
               |                              |
               |                              v
               |                    AWS provider plugin
               |                              |
               v                              v
      Remote state backend <---------> AWS service APIs
      S3 + encryption/versioning       VPC, IAM, EKS, RDS...
      + native S3 lockfile
```

一次受控變更的流程：

1. `terraform init` 初始化 backend，下載 configuration 所需的 providers 與 modules。
2. `terraform validate` 驗證 syntax 和內部一致性；它不證明 AWS 架構安全或可運作。
3. `terraform plan` refresh 已管理 resources、比較 configuration 與 state，建立 proposed actions。
4. Reviewer 檢查 create/update/replace/destroy、unknown values、IAM/network/data risk 與預期外變更。
5. `terraform apply tfplan` 執行同一個已保存且已核准的 plan；provider 將 graph nodes 轉換成 AWS API calls。
6. Terraform 在成功操作後更新 state。Apply 完成仍要用 AWS APIs、application health 和 network path 驗證 effective behavior。

Terraform 會平行處理沒有 dependency 的 resources。Dependency 通常由 expression reference 自動推導，例如 `subnet_id = aws_subnet.app.id`。只在沒有 data reference、但確實存在 behavioral dependency 時使用 `depends_on`；過度使用會增加 unknown values 並降低 concurrency。

## Configuration、state 與 real infrastructure

| 元件 | 角色 | 不是什麼 |
| --- | --- | --- |
| Configuration | Git 中的 desired infrastructure 與 module composition | 目前 AWS inventory |
| State | Terraform address 與 remote object identity/attributes 的 mapping | 可任意手改的 database 或完整 backup |
| Remote infrastructure | AWS APIs 回報的 effective resources | Configuration 改變後會自動變更的環境 |
| Plan | 特定 configuration、state、credentials 與時間點的 proposed actions | 永遠有效的承諾 |

State 讓 Terraform 知道 `aws_vpc.main` 對應哪個 VPC、追蹤 dependency 並處理 resource lifecycle。不要直接編輯 JSON state；使用 `terraform state`、`import`、`moved` 等受支援機制，高風險 state operation 前保留可驗證 backup。

`sensitive = true` 主要是遮蔽 CLI/UI 顯示，不代表值不會進入 state 或 saved plan。State、plan artifacts 與 crash logs 都可能包含 secrets，不能提交 Git，也不能無限制保存。

## AWS production state architecture

```hcl
terraform {
  backend "s3" {
    bucket       = "org-terraform-state-prod"
    key          = "network/prod/terraform.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Production backend 應具備：

- S3 Block Public Access、server-side encryption、bucket versioning、TLS-only policy、audit strategy，以及測試過的 recovery procedure。
- `use_lockfile = true` 啟用 S3 native state locking，避免 concurrent writers。DynamoDB-based locking 已 deprecated，只應在舊版本 migration 過渡期保留，不應成為新架構預設。
- Backend bucket、state object 與 `.tflock` 的 least-privilege IAM permissions。Human state read 應是 exception；視風險分離 read-only plan role 與 pipeline write role。
- CI 透過 OIDC/federation assume short-lived IAM role；不在 HCL、`-backend-config`、`.tfvars` 或 CI variables 保存 long-lived access keys。
- 依 environment、account、Region、team ownership 與 blast radius 切分 state。不要用一個巨大 state 同時管理 organization、network、EKS、databases 與 applications。
- Backend 是 bootstrap dependency，通常由受限制的獨立 foundation process 建立；不能靠尚未存在的 backend state 建立自己。

Lock 只防止相同 backend state 的 concurrent mutation，不能阻止另一個 state、CloudFormation、console user 或 controller 修改同一 AWS resource。因此每個 resource 必須只有一個 lifecycle owner。

## Repository、root module 與 reusable modules

```text
infrastructure/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── tests/
│   └── eks/
├── live/
│   ├── dev/network/
│   ├── dev/eks/
│   ├── prod/network/
│   └── prod/eks/
└── policies/
```

- **Root module** 組合 provider、backend、environment inputs 與 child modules，是 plan/apply 和 state 的 operational boundary。
- **Reusable module** 封裝有意義的 architecture capability，例如 secure VPC 或 EKS cluster；不要替每個單一 resource 建立 thin wrapper。
- 保持 module tree 扁平。由 root module 把 `module.network` outputs 傳給 `module.eks`，比深層 module nesting 更容易理解、測試與替換。
- Child module 宣告 `required_providers`，但 provider configuration 應由 root module 管理，必要時透過 aliases 傳入。
- Inputs/outputs 要有 description、精確 type 和 validation；重要 assumptions/guarantees 可用 `precondition`、`postcondition` 或 `check` 表達。
- Shared modules 使用 immutable version 或 commit SHA，透過 tests、changelog 和 upgrade notes 管理 breaking changes。
- `.terraform.lock.hcl` 必須提交 version control。它鎖定 provider selection；remote modules 仍須明確設定 version/ref。

`dev` 與 `prod` 應使用不同 state，最好也分離 AWS accounts 和 deployment roles。Terraform CLI workspaces 可建立相同 configuration 的不同 state instances，但相同 code/backend access 不是強 isolation boundary；當 credentials、approval 或 architecture 不同時，使用獨立 root modules、states、accounts 與 roles。HCP Terraform workspace 有更完整的 execution/state/policy capabilities，不能只因名稱相同就與 CLI workspace 視為完全相同。

## Version 與 provider 管理

```hcl
terraform {
  required_version = ">= 1.10, < 2.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = var.owner
    }
  }
}
```

版本只是範例，不應不經驗證直接複製到 production。Root module 設定已測試 constraint 並提交 lock file；reusable child module 通常宣告相容 range，避免不必要限制 caller。Upgrade 用獨立 PR，閱讀 changelog、更新 lock file、執行 module tests，再逐 environment rollout。

不要在 provider block 放 static credentials。AWS provider 使用 standard credential chain、OIDC、role assumption 或 workload identity。Cross-account deployment 可由 central runner assume environment-specific role；每個 account/Region alias 都要有清楚 ownership 和 least privilege。

## CI/CD delivery architecture

```text
Pull request
  -> fmt -check / validate / lint
  -> secrets + IaC security + policy checks
  -> terraform test (適用時)
  -> speculative plan + readable summary
  -> peer review
Merge / protected deployment
  -> fresh final plan
  -> production approval
  -> apply the saved final plan
  -> AWS/runtime verification
  -> evidence, notification, drift monitoring
```

- PR plan 是 review evidence；apply 前仍產生 fresh final plan，避免 stale plan 或期間發生 drift。
- Apply 使用 reviewed saved plan，不在 approval 後重新執行會產生另一份未審核 plan 的 `terraform apply -auto-approve`。
- 同一 state 的 jobs 使用 pipeline concurrency control。Backend locking 是最後一道保護，不是完整 workflow queue。
- 保護 production branch、runner、environment、role 和 approval。Plan job 不應自然擁有 destroy、IAM escalation 或 production apply 的全部權限。
- 提供 reviewer 可讀的 create/change/replace/destroy summary，但 binary/JSON plan 可能含 secrets，artifact 必須限制存取、縮短 retention 並禁止洩漏到 public logs。
- Gates 包含 `fmt`、`validate`、lint、secret/IaC scan、policy as code、module tests 及 cost/security review；工具通過不代表 architecture 已安全。
- Apply success 只表示 provider operations 成功並寫入 state。EKS、network、IAM、database 仍需 domain-specific runtime verification。

## Security 與 governance

- Execution role 使用 short-lived credentials、least privilege，並分離 production/non-production；適用時加入 permission boundaries 或 SCPs。
- IAM、KMS、public exposure、security groups、routes、data deletion 與 replacement 應有額外 review gates。
- Secrets 放 Secrets Manager 或 Parameter Store，讓 workload runtime 讀取。若 Terraform 產生或讀取 secret，內容仍可能進入 state。
- Policy as code 提供 preventive guardrails；static scanning 找 HCL misconfiguration；AWS Config、Security Hub、GuardDuty 等 detective controls 驗證 deployed reality。
- External modules 是 supply-chain dependency：review source、maintainer、release、transitive behavior 和 license，並 pin immutable version。
- `prevent_destroy` 可保護 critical resource，但不是 backup，也可能阻擋 recovery。仍需 service-native deletion protection、backup、retention 和 break-glass runbook。

## Terraform、Helm 與 Argo CD ownership

| Layer | 建議 owner |
| --- | --- |
| AWS account foundation、VPC、IAM、EKS control plane、node roles、KMS | Terraform |
| Kubernetes add-on 的 AWS prerequisites | Terraform |
| Namespaced workloads、Deployments、Services、ConfigMaps、application charts | Helm/Argo CD |
| Git desired state 與 continuous Kubernetes reconciliation | Argo CD |

Terraform 能使用 Kubernetes/Helm providers，但不代表所有 cluster objects 都應由 Terraform 管理。若建立 EKS cluster 後，在相同 state 立刻使用 Kubernetes provider，planning、credentials、cluster availability 與 destroy ordering 會高度耦合。通常應分離 infrastructure state 與 cluster add-on/application delivery。

不要讓 Terraform 與 Argo CD 同時管理相同 Kubernetes object，否則兩個 reconciliation loops 會互相覆寫。Ownership transition 應先 inventory 並 match live state，再停用舊 writer，最後啟用新 writer。

## Drift、import 與 safe refactoring

- 定期執行 read-only plan 檢查 drift，將 unexpected change 分派給 owner。Incident-time console fix 應先評估，再寫回 code 或明確 revert。
- 對既有 infrastructure 使用 declarative `import` block 或受控 CLI import。Import 只建立 state mapping；first post-import plan 應為 no-op 或只有逐項說明且核准的差異。
- Rename resource address 或移動 module 時使用 `moved` block，保留 identity，避免 destroy/recreate。
- 若保留 remote resource 但停止 Terraform ownership，使用受支援的 removed/state workflow，不直接刪除 state JSON。
- `ignore_changes` 只用於明確 shared attribute，並記錄另一個 owner。大範圍 ignore 會隱藏 drift。
- `-target` 是 exceptional recovery/troubleshooting tool，不是日常 deployment strategy；routine targeting 可能跳過 graph 需要的變更。

## Failure、rollback 與 recovery

Terraform 沒有 application-style automatic rollback。Apply 可能部分成功：已成功的 AWS operations 會寫入 state，下一次 plan 顯示剩餘差異。

1. 停止 concurrent writers，保留 logs、plan 和 current state evidence。
2. 用 AWS APIs 確認實際成功的 operations，不只看 pipeline 最後一行。
3. 修正 root cause 後重新 plan。通常選擇 forward fix；revert 舊 configuration 也必須重新 plan 並評估 data/replacement。
4. 只有 state 遺失或損壞時才考慮 S3 version recovery。還原舊 state 不會回復 AWS infrastructure，反而可能產生危險 plan。
5. Data-bearing resources 必須依靠 service-native backup、snapshot、replication 和 tested restore；Terraform state 不是資料備份。

常見失敗包括 stale lock、credential expiration、AWS throttling/quota、eventual consistency、provider bug、manual drift、resource already exists，以及 replacement 受 data protection 阻擋。`force-unlock` 前必須證明沒有 active writer 並核對 lock ID。

## 常見面試題的精準回答

### Terraform 如何決定 resource 順序？

Terraform 從 references 建立 directed dependency graph，沒有依賴的 nodes 可平行執行。`depends_on` 只處理 data flow 無法表達的 hidden dependency；檔案順序不控制 resource 順序。

### State 是 source of truth 嗎？

Configuration 表達 intent，AWS APIs 是 real infrastructure，state 保存 Terraform address 與 remote identity/attributes 的 mapping；plan 結合三者。把 state 單獨稱為唯一 source of truth 會忽略 drift 和 desired configuration。

### 為什麼 production 需要 remote state 與 locking？

Remote backend 提供 shared、access-controlled 且可復原的 state；locking 防止相同 state concurrent writers。S3 應啟用 encryption、versioning 和 native `use_lockfile`。Lock 不會阻止 console 或其他 IaC state 修改同一 resource。

### `plan` 成功是否代表變更安全？

不是。Plan 只是特定時間和 credentials 下的 proposed diff。仍要判斷 replacement、data loss、IAM/network blast radius、unknown values、quota、runtime dependency 與 recovery；apply 後還要驗證 effective behavior。

### Module 應如何切分？

依有意義的 architecture capability 和 lifecycle ownership 切分，不是每個 resource 一個 wrapper。保持 flat composition、small typed interfaces，讓 root module 負責 providers 與 environment wiring。

### Workspaces 是否等於 environment isolation？

CLI workspaces 提供相同 configuration 的多個 state instances，但通常共享 code/backend access，不等於 account、credential 或 approval isolation。嚴格邊界使用 separate accounts、roles、root modules 和 states。

### 如何安全接管既有 AWS resource？

先 inventory effective settings，寫出 matching configuration，再 import identity；first plan 必須 no-op 或只有已核准差異。不要同時做 ownership migration、major upgrade 和 redesign，也要先停止原 writer。

### Apply 失敗時是否 restore 舊 state？

通常不應。先確認 remote reality，修正問題並重新 plan/forward-fix。舊 state 不是 infrastructure snapshot；只有 state 本身遺失或損壞時才執行受控 recovery。

### Terraform 與 Argo CD 如何分工？

Terraform 管理 AWS infrastructure 和 cluster prerequisites；Argo CD 管理 Git 中的 Kubernetes desired state 並持續 reconcile。核心原則是一個 object 只有一個 lifecycle owner。

## Review checklist

- [ ] 每個 root/state 有明確 environment、account、owner 與 blast radius。
- [ ] S3 state 啟用 encryption、versioning、Block Public Access 與 native lockfile。
- [ ] CI 使用 short-lived role，並依 sensitive data 保護 state、plans、logs 和 artifacts。
- [ ] Terraform、providers、modules 和 `.terraform.lock.hcl` 有受控 version strategy。
- [ ] Modules 表達 architecture capabilities，保持 flat composition 並具備 tests/documentation。
- [ ] PR plan、production approval、saved-plan apply 和 post-apply verification 形成同一 evidence chain。
- [ ] Imports、moved resources、drift、break-glass changes 與 ownership transfer 有 runbook。
- [ ] Data-bearing/destructive change 有 service-native backup 和 tested recovery evidence。
- [ ] Terraform、CloudFormation、operators、Helm 與 Argo CD 不會管理相同 object。

## 參考資料

- [Terraform core workflow](https://developer.hashicorp.com/terraform/intro/core-workflow)
- [Terraform style guide](https://developer.hashicorp.com/terraform/language/style)
- [State storage and locking](https://developer.hashicorp.com/terraform/language/state/backends)
- [S3 backend and native locking](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- [Module composition](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
- [Terraform test](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [AWS Provider best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/introduction.html)
- [AWS Terraform security best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/security.html)
- [AWS Terraform code structure](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/structure.html)

---

# Terraform Architecture and Best Practices: Interview-Depth Guide

Terraform is a declarative Infrastructure as Code (IaC) tool. You describe desired infrastructure; Terraform reads configuration, state, and remote objects returned by providers, constructs a dependency graph and execution plan, and invokes provider APIs to converge the real environment.

This guide belongs in **AWS Cloud** because Terraform primarily manages cloud infrastructure such as VPC, IAM, EKS, databases, and DNS. It is not part of Kubernetes. Pipeline scanning, approvals, and credentials are DevSecOps controls, but DevSecOps is not the right category for the overall architecture guide.

## Architecture and execution flow

```text
Git repository / reusable modules
               |
               v
      Terraform CLI or remote runner
        | configuration + variables
        | provider schemas
        | prior state + refreshed remote state
        v
       Dependency graph -> execution plan -> apply
               |                              |
               |                              v
               |                    AWS provider plugin
               |                              |
               v                              v
      Remote state backend <---------> AWS service APIs
      S3 + encryption/versioning       VPC, IAM, EKS, RDS...
      + native S3 lockfile
```

A controlled change follows this sequence:

1. `terraform init` initializes the backend and installs required providers and modules.
2. `terraform validate` checks syntax and internal consistency; it does not prove the AWS design is secure or operable.
3. `terraform plan` refreshes managed resources, compares configuration with state, and proposes actions.
4. Reviewers inspect creates, updates, replacements, destroys, unknown values, and IAM, network, and data risks.
5. `terraform apply tfplan` executes the same saved and approved plan; providers translate graph nodes into AWS API calls.
6. Terraform updates state after successful operations. Verify effective behavior through AWS APIs, application health, and relevant network paths.

Terraform runs graph nodes without dependencies in parallel. References normally imply dependencies. Use `depends_on` only for a real behavioral dependency that data flow cannot express; excessive explicit dependencies increase unknown values and reduce concurrency.

## Configuration, state, and real infrastructure

| Component | Role | What it is not |
| --- | --- | --- |
| Configuration | Desired infrastructure and module composition stored in Git | Current AWS inventory |
| State | Mapping between Terraform addresses and remote identities/attributes | A database to edit manually or a complete backup |
| Remote infrastructure | Effective resources returned by AWS APIs | Automatically changed when configuration changes |
| Plan | Proposed actions for specific configuration, state, credentials, and time | A permanently valid promise |

State maps `aws_vpc.main` to a specific VPC and tracks dependency and lifecycle information. Do not edit state JSON directly. Use supported `terraform state`, `import`, and `moved` workflows, retaining a verified backup before high-risk operations.

Marking a value `sensitive` primarily redacts output. The value can still exist in state or a saved plan. Treat state, plan artifacts, and crash logs as sensitive data; never commit them to Git or retain them without controls.

## AWS production state architecture

```hcl
terraform {
  backend "s3" {
    bucket       = "org-terraform-state-prod"
    key          = "network/prod/terraform.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

A production backend needs:

- S3 Block Public Access, server-side encryption, versioning, a TLS-only policy, an audit strategy, and tested recovery.
- Native S3 locking through `use_lockfile = true`. DynamoDB-based locking is deprecated and should remain only temporarily during migration from older Terraform versions.
- Least-privilege access to the bucket, state key, and `.tflock`. Human state reads should be exceptional; separate read-only plan and pipeline write roles where warranted.
- Short-lived CI credentials through OIDC/federation and role assumption. Never place long-lived keys in HCL, backend arguments, `.tfvars`, or CI variables.
- State boundaries based on environment, account, Region, ownership, and blast radius. Avoid one state for the organization, network, EKS, databases, and applications.
- An explicit bootstrap process because the backend must exist before a configuration can use it.

Locking only prevents concurrent mutation of the same backend state. It cannot stop another state, CloudFormation stack, console user, or controller from modifying the same AWS object. Every resource needs one lifecycle owner.

## Repository, root module, and reusable modules

```text
infrastructure/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── tests/
│   └── eks/
├── live/
│   ├── dev/network/
│   ├── dev/eks/
│   ├── prod/network/
│   └── prod/eks/
└── policies/
```

- A **root module** composes providers, backend, environment inputs, and child modules. It is the plan/apply and state boundary.
- A **reusable module** encapsulates a meaningful architectural capability such as a secure VPC or EKS cluster. Avoid thin wrappers around single resources.
- Keep the module tree flat. Passing network outputs into an EKS module from the root is easier to understand, test, and replace than deep nesting.
- Child modules declare `required_providers`; root modules own provider configurations and pass aliases where needed.
- Give inputs and outputs descriptions, precise types, and validation. Express important assumptions and guarantees with conditions or checks.
- Pin shared modules to immutable versions or commits and manage breaking changes with tests, changelogs, and upgrade notes.
- Commit `.terraform.lock.hcl`. It locks provider selection; remote module versions or references must still be constrained explicitly.

Development and production need separate state and preferably separate AWS accounts and roles. CLI workspaces create multiple state instances for one configuration, but shared code and backend access are not strong isolation. Use separate roots, state, accounts, and roles when credentials, approvals, or architecture differ. HCP Terraform workspaces provide broader execution, state, and policy capabilities and are not identical to CLI workspaces.

## Version and provider management

```hcl
terraform {
  required_version = ">= 1.10, < 2.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = var.owner
    }
  }
}
```

Versions are examples, not values to copy into production without validation. Root modules constrain tested versions and commit the lock file. Reusable modules normally declare compatible ranges. Upgrade in dedicated pull requests, read changelogs, update locks, run tests, and roll through environments progressively.

Do not put static credentials in provider blocks. Use the AWS credential chain, OIDC, role assumption, or workload identity. A central runner can assume environment-specific roles; each account and Region alias needs explicit ownership and least privilege.

## CI/CD delivery architecture

```text
Pull request
  -> fmt -check / validate / lint
  -> secrets + IaC security + policy checks
  -> terraform test where appropriate
  -> speculative plan + readable summary
  -> peer review
Merge / protected deployment
  -> fresh final plan
  -> production approval
  -> apply the saved final plan
  -> AWS/runtime verification
  -> evidence, notification, drift monitoring
```

- A PR plan is review evidence. Generate a fresh final plan before deployment to detect intervening drift.
- Apply the reviewed saved plan. Do not approve one plan and then let `terraform apply -auto-approve` calculate an unreviewed replacement.
- Serialize jobs for the same state. Backend locking is the last concurrency guard, not a workflow queue.
- Protect the production branch, runner, environment, role, and approval path. A plan job should not automatically receive destructive or privilege-escalation permissions.
- Provide a readable action summary, but protect binary and JSON plans because they can contain secrets. Restrict artifacts, shorten retention, and keep secrets out of public logs.
- Run formatting, validation, lint, secret/IaC scans, policy checks, module tests, and cost/security review. Tool success does not prove architecture safety.
- Apply success proves provider operations completed and state was written. EKS, networks, IAM, and databases still need domain-specific runtime verification.

## Security and governance

- Use short-lived, least-privilege execution roles and separate production from non-production. Add permission boundaries or SCPs where appropriate.
- Require extra review for IAM, KMS, public exposure, security groups, routes, deletion, and replacement of data-bearing resources.
- Store application secrets in Secrets Manager or Parameter Store and retrieve them at runtime. Terraform-generated or read secrets may still enter state.
- Use policy as code for preventive guardrails, static scanning for HCL mistakes, and AWS Config, Security Hub, or GuardDuty for deployed reality.
- Treat external modules as supply-chain dependencies: review source, maintainers, releases, transitive behavior, and licenses, and pin immutable versions.
- `prevent_destroy` can protect selected resources, but it is not a backup and can impede recovery. Combine it with service deletion protection, backups, retention, and a break-glass runbook.

## Ownership between Terraform, Helm, and Argo CD

| Layer | Recommended owner |
| --- | --- |
| AWS account foundation, VPC, IAM, EKS control plane, node roles, KMS | Terraform |
| AWS prerequisites for Kubernetes add-ons | Terraform |
| Namespaced workloads, Deployments, Services, ConfigMaps, application charts | Helm/Argo CD |
| Git desired state and continuous Kubernetes reconciliation | Argo CD |

Terraform can use Kubernetes and Helm providers, but every cluster object need not be in Terraform. Creating EKS and immediately using the Kubernetes provider in the same state couples planning, credentials, cluster availability, and destroy ordering. Prefer separating infrastructure state from add-on and application delivery.

Never let Terraform and Argo CD manage the same Kubernetes object. During ownership migration, inventory and match live state, stop the old writer, and then enable the new writer.

## Drift, import, and safe refactoring

- Run scheduled read-only plans and route unexpected drift to the owner. Assess emergency console changes before writing them into code or reverting them.
- Adopt infrastructure with declarative `import` blocks or controlled CLI import. Import establishes a state mapping; the first plan must be a no-op or contain only explained and approved differences.
- Use `moved` blocks when renaming addresses or reorganizing modules so identity is retained rather than replaced.
- Use supported removed/state workflows when retaining a remote object but ending Terraform ownership. Never delete state JSON manually.
- Use `ignore_changes` only for an explicitly shared attribute and document the other owner. Broad ignores hide drift.
- Treat `-target` as an exceptional recovery tool, not normal deployment; routine targeting can skip necessary graph changes.

## Failure, rollback, and recovery

Terraform has no application-style automatic rollback. An apply can partially succeed; completed operations are written to state, and the next plan reports what remains.

1. Stop concurrent writers and retain logs, plan, and current-state evidence.
2. Query AWS APIs to determine which operations actually succeeded.
3. Fix the root cause and plan again. Prefer a forward fix; reverting configuration still requires a new plan and data/replacement assessment.
4. Restore an S3 state version only when state itself is lost or corrupt. Restoring old state does not roll back AWS and can produce a dangerous plan.
5. Protect data with service-native backups, snapshots, replication, and tested restores. Terraform state is not a data backup.

Common failures include stale locks, credential expiry, throttling, quota exhaustion, eventual consistency, provider defects, drift, existing resources, and replacement blocked by data protection. Before `force-unlock`, prove no writer is active and verify the lock ID.

## Precise interview answers

### How does Terraform determine resource order?

Terraform builds a directed dependency graph from references and runs independent nodes in parallel. Use `depends_on` only for hidden dependencies that data flow cannot express. File order does not control resource order.

### Is state the source of truth?

Configuration expresses intent, AWS APIs expose real infrastructure, and state maps Terraform addresses to remote identities and attributes. Planning combines all three. Calling state the sole source of truth ignores drift and desired configuration.

### Why do production systems need remote state and locking?

Remote state is shared, access-controlled, and recoverable. Locking prevents concurrent writers to the same state. On S3, enable encryption, versioning, and native `use_lockfile`. Locking does not prevent console or other IaC systems from changing a resource.

### Does a successful plan mean the change is safe?

No. A plan is a proposed diff for a particular moment and credentials. Review replacement, data loss, IAM/network blast radius, unknowns, quota, runtime dependencies, and recovery, then validate effective behavior after apply.

### How should modules be divided?

Divide by meaningful architectural capability and lifecycle ownership, not one wrapper per resource. Keep composition flat and interfaces small and typed; let the root module wire providers and environments.

### Are workspaces environment isolation?

CLI workspaces provide separate state instances for one configuration but often share code and backend access. Strong isolation uses separate accounts, roles, roots, and state.

### How do you safely adopt an existing AWS resource?

Inventory effective settings, write matching configuration, and import the identity. Require a no-op or fully explained first plan. Do not combine ownership migration with a major upgrade or redesign, and stop the previous writer first.

### Should you restore old state after a failed apply?

Usually not. Confirm remote reality, correct the failure, and re-plan for a forward fix. Old state is not an infrastructure snapshot. Recover state only when state itself is lost or corrupted.

### How do Terraform and Argo CD divide responsibility?

Terraform owns AWS infrastructure and cluster prerequisites. Argo CD owns Kubernetes desired state and continuous reconciliation. One object must have one lifecycle owner.

## Review checklist

- [ ] Every root/state has a clear environment, account, owner, and blast radius.
- [ ] S3 state uses encryption, versioning, Block Public Access, and native locking.
- [ ] CI uses short-lived roles and protects state, plans, logs, and artifacts as sensitive data.
- [ ] Terraform, providers, modules, and `.terraform.lock.hcl` follow a controlled version strategy.
- [ ] Modules represent architectural capabilities, use flat composition, and have tests/documentation.
- [ ] PR plan, production approval, saved-plan apply, and post-apply validation form one evidence chain.
- [ ] Imports, moved resources, drift, break-glass changes, and ownership transfers have runbooks.
- [ ] Destructive or data-bearing changes have service-native backup and tested recovery evidence.
- [ ] Terraform, CloudFormation, operators, Helm, and Argo CD do not manage the same object.

## References

- [Terraform core workflow](https://developer.hashicorp.com/terraform/intro/core-workflow)
- [Terraform style guide](https://developer.hashicorp.com/terraform/language/style)
- [State storage and locking](https://developer.hashicorp.com/terraform/language/state/backends)
- [S3 backend and native locking](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- [Module composition](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
- [Terraform test](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [AWS Provider best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/introduction.html)
- [AWS Terraform security best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/security.html)
- [AWS Terraform code structure](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/structure.html)
