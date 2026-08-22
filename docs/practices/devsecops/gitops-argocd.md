# Argo CD GitOps 最佳實務

## 核心觀念

GitOps 將 Git 中經過審查的宣告式設定視為 Kubernetes 的期望狀態
（desired state）。Argo CD 持續比較期望狀態與叢集的實際狀態（live
state），顯示差異，並依同步政策進行修正。

這不代表要移除 CI：

- CI 負責建置、測試、掃描、簽章與發布 immutable artifact。
- CI 或發布流程以 Pull Request 更新設定儲存庫中的 image digest、Chart
  版本或 values。
- Argo CD 負責從 Git 拉取期望狀態，產生 manifest，並持續 reconcile
  Kubernetes。
- 監控與驗證負責證明新版本真的能服務流量；`Synced` 本身不是發布成功的
  完整證據。

```text
Application repository                 Configuration repository
source -> CI -> image@sha256:digest -> PR -> review -> merge
                                                    |
                                                    v
Git webhook / polling -> Argo CD -> render -> diff -> sync -> Kubernetes
                                      ^                         |
                                      |                         v
                            desired state              live state + health
```

## 為什麼偏好 GitOps-driven CD

| 面向 | Pipeline-driven deployment | GitOps-driven deployment |
| --- | --- | --- |
| 執行模型 | CI job 執行一次 `helm upgrade` 或 `kubectl apply` 後結束 | Controller 持續比較 Git 與 live state |
| Drift | 除非另做稽核，手動變更可能長期存在 | 顯示 `OutOfSync`，可選擇自動 self-heal |
| 稽核 | 必須串接 pipeline、參數與執行紀錄 | Git commit、review、diff 與 Argo operation 形成可追溯鏈 |
| 權限 | CI runner 通常需要叢集寫入權限 | CI 只需更新 Git；叢集憑證留在 CD control plane |
| 回復 | 重跑歷史 job 與參數，容易重建出不同 artifact | Revert Git commit，再由 controller 收斂 |
| 狀態可見性 | Job 成功不代表後續狀態仍正確 | 持續呈現 sync status、operation status 與 health |

GitOps 並非在所有情況都比較好。一次性維運動作、資料庫內部 migration、外部
SaaS API 呼叫或 incident break-glass 操作仍可能需要 imperative workflow。
重點是明確指定 ownership，避免 CI、Argo CD、Terraform 與人工 `kubectl`
同時寫入同一個欄位或資源。

## Argo CD 架構

```text
Developers / CI / SSO
         |
         v
+-------------------+       webhook       +-------------------+
| Argo CD API Server| <------------------- | Git provider      |
| UI, CLI, API, RBAC|                      +-------------------+
+---------+---------+
          |
          | application operations
          v
+---------+----------+   manifest request  +-------------------+
| Application        | ------------------> | Repository Server |
| Controller          | <------------------ | clone/cache/render|
| compare/reconcile   |    rendered YAML   +-------------------+
+---------+----------+
          |
          | Kubernetes API
          v
+--------------------+
| Managed cluster(s) |
+--------------------+
```

- **API Server**：提供 UI、CLI、gRPC/REST API、authentication、RBAC、
  repository/cluster credential 管理及 webhook 接收。
- **Repository Server**：clone/cache Git source，並使用 Helm、Kustomize、
  Jsonnet 或 plain YAML 產生 manifest。因為它會處理不受信任的輸入，應限制
  網路、權限及自訂 plugin。
- **Application Controller**：watch `Application` 與 managed resources，
  比較 target/live state，更新 status，並執行 sync、hook 與修正動作。
- **Redis**：快取 repository、manifest 與 application state，降低 Git 與
  Kubernetes API 負載；不是期望狀態的 source of truth。
- **ApplicationSet Controller**：由 cluster、Git directory、list 等 generator
  產生多個 `Application`，適合重複環境或多叢集，但 generator/template
  錯誤也可能放大 blast radius。

Argo CD Core 是沒有常駐 API Server、OIDC、Argo CD RBAC 與 notification
controller 的 headless 安裝模式，主要依賴 Kubernetes RBAC。一般多人平台若需要
SSO、UI、multi-tenancy 與集中治理，通常使用完整安裝。

## 核心資源與狀態

- `Application`：定義一組資源的 source、revision、path/chart、destination
  與 sync policy。
- `AppProject`：限制 Application 可使用的 source repositories、目的 cluster/
  namespace 及允許的 resource kinds，是 multi-tenancy 與 blast-radius 邊界。
- `ApplicationSet`：用 template 加 generator 建立多個 Applications；它不是
  Application 內部 workload 的 deployment controller。
- **Refresh**：重新取得 target 與 live state 並計算差異，不等於 sync。
- **Sync status**：`Synced` 表示 live state 與 Git render 結果一致；
  `OutOfSync` 表示存在差異。
- **Operation status**：最近一次 sync 是否成功。
- **Health status**：Deployment 是否 available、Pod 是否 ready 等資源健康判斷。

因此可能出現 `Synced` 但 `Progressing` 或 `Degraded`。面試或事故分析時，應分開
回答「設定是否已收斂」與「服務是否可用」。

## Repository 與 artifact 設計

- 當 application code 與環境設定有不同權限、owner 或 release cadence 時，使用
  獨立 configuration repository。這可避免單純修改 replicas 也觸發 rebuild，並
  提供乾淨的 production audit history。
- 使用 immutable reference：container image digest、Helm Chart version、Git
  commit SHA，以及固定版本的 remote Kustomize base。浮動 tag 或 branch 可能讓
  相同 Git revision 在不同時間 render 出不同 manifest。
- 不要把 plaintext secret 放進 Git。使用 External Secrets Operator、Secrets
  Store CSI Driver、SOPS 等合適機制，並限制解密權限與 secret exposure。
- 對由其他 controller 管理的欄位保留 imperativeness。例如 HPA 管理 replicas
  時，不要讓 Git 同時固定 `spec.replicas`。
- 避免以 Argo CD parameter override 取代 Git commit；若變更只存在於 Argo CD
  API 中，Git 就不再是完整 source of truth。

## Application 邊界與 ownership

Application 應依 lifecycle、owner、risk 與 rollback unit 劃分，而不只是依
namespace 或 repository 劃分：

- 將 application workload、shared platform controller 與 stateful data service
  分開，避免一般應用發布意外變更資料庫或 ingress control plane。
- 同一 Kubernetes object 只交給一個 reconciler 管理。先辨識 Terraform、Helm
  release、operator-generated child、CI job 與人工流程的現有 ownership。
- Controller 產生的 child resources 通常應由 controller 管理，不要再放進另一個
  Argo Application。
- 用 AppProject allowlist 限制 repository、destination 與 resource kind；不要讓
  每個 project 預設取得任意 cluster-scoped resource 權限。
- App-of-apps 或 ApplicationSet 的父層可以建立高權限 Application，只有平台管理者
  應能修改其 source 與 template。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  sourceRepos:
    - https://git.example.com/platform/payments-config.git
  destinations:
    - namespace: payments
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-production
  namespace: argocd
spec:
  project: payments
  source:
    repoURL: https://git.example.com/platform/payments-config.git
    targetRevision: 4d2c0f6
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    syncOptions:
      - CreateNamespace=false
```

上例刻意不先啟用 automated sync、prune 或 self-heal。真正的 production policy
應再依平台需求加入 RBAC allowlist/denylist、sync window 與審批流程；範例不是完整
的 security baseline。

## 安全導入現有資源

1. **盤點**：記錄 live manifest、UID、Service ClusterIP、PVC、credentials、
   generated children、field managers、health 與所有現有 writer。
2. **建立相同期望狀態**：從目前實際版本 render manifest，不要在 adoption 同時做
   upgrade、rename 或架構重整。
3. **限制範圍**：建立最小權限 AppProject，以及 manual、non-pruning、沒有
   self-heal 的 Application。
4. **先看 diff**：區分預期差異、API defaulting、mutating webhook、controller
   欄位與真正 drift。不要用廣泛 `ignoreDifferences` 隱藏未知問題。
5. **小批同步**：先同步一個低風險單元，驗證 rollout、traffic、identity、storage、
   data 與外部 side effects，而不只看 `Synced`。
6. **退休舊 writer**：停用原本的 pipeline deploy、Helm automation 或人工 SOP，
   確保同一資源只有 Argo CD 寫入。
7. **逐步自動化**：觀察穩定後先評估 self-heal，再針對已盤點的 resource 評估
   prune。`allowEmpty` 與 cascading deletion 必須有額外保護。
8. **演練復原**：驗證 Git revert、failed sync、controller outage、credential
   rotation、Argo CD backup/restore 與 break-glass 後的 drift reconciliation。

對 stateful workload，還要保留 PVC identity、Service identity、membership、
quorum、backup 與 restore proof。Git revert 可以還原 manifest，但不能自動還原已
執行的 database migration 或遺失的資料。

## 自動同步、順序與發布

- `automated` 讓 Argo CD 在 target/live state 不一致時同步；`selfHeal` 允許修正
  live drift；`prune` 允許刪除 Git 已移除的 tracked resources。三者風險不同，
  應分開決策。
- Sync phases 與 resource hooks 適合 PreSync migration、Sync workload、PostSync
  驗證；hooks 必須 idempotent，並有 timeout、重試與失敗處理。
- Sync waves 可排序資源套用，但不能取代 dependency readiness、schema backward
  compatibility 或 application-level health check。
- 使用 sync windows 保護 freeze period 或限制 production change 時段，但仍需設計
  emergency override 與 audit trail。
- Rollback 優先以 Git revert 建立新的可審核 desired-state commit。對 database
  schema、PVC、CRD 與 external side effects，必須有獨立且經演練的復原方案。

## Security 與營運檢查表

- [ ] SSO/OIDC、Argo CD RBAC 與 Kubernetes RBAC 採 least privilege，管理者與
  application team 職責分離。
- [ ] AppProject 限制 source、destination、cluster-scoped kinds 與 namespace
  resources。
- [ ] Repository、cluster 與 secret-store credentials 可輪替、不寫入 manifest，
  並限制 repository server 的 exposure。
- [ ] Git branch protection、required review、signed commit/tag 或 provenance 符合
  supply-chain 風險。
- [ ] Image、Chart、Git dependency 與 remote base 都使用 immutable version。
- [ ] Argo CD、其 CRDs 與 dependencies 有固定版本、upgrade rehearsal、backup 與
  disaster-recovery runbook。
- [ ] 監控 controller queue/reconciliation、Git/Kubernetes API errors、cache、
  `OutOfSync`、`Unknown`、`Degraded` 與長時間 `Progressing`。
- [ ] Notifications 有明確 owner；告警能連到 commit、diff、operation 與 runbook。
- [ ] Break-glass 變更有時限與稽核，事件後回寫 Git 或由 self-heal 還原。
- [ ] 驗證鏈涵蓋 commit、artifact digest、rendered manifest、sync result、rollout、
  traffic、SLO 與資料完整性。

## 面試重點

**Argo CD 與 GitLab CI 的責任如何切分？**  
CI 建置與驗證一次 immutable artifact，透過 PR 更新環境設定；Argo CD 持續執行
CD reconciliation。CI 不應再同時直接部署同一組 Argo-managed resources。

**`Synced` 是否代表部署成功？**  
不代表。它只證明 live state 與 rendered target state 一致；還要看 operation、
health、rollout、traffic、SLO、migration 與外部依賴。

**為什麼不在第一天開啟 prune 與 self-heal？**  
既有叢集可能有未盤點資源、另一個 writer、defaulted/mutated fields 或錯誤的
Application scope。先 manual/non-pruning adoption，證明 diff 與 ownership 後再逐步
自動化，可控制刪除與覆寫的 blast radius。

**AppProject 與 Kubernetes RBAC 有何不同？**  
AppProject 約束 Argo Application 可以從哪裡部署到哪裡以及可管理哪些 kinds；
Kubernetes RBAC 約束 Argo components 與使用者在 Kubernetes API 上能做什麼。兩層
都要最小權限，不能互相取代。

## 參考資料

- Argo CD, [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- Argo CD, [Understand the Basics](https://argo-cd.readthedocs.io/en/stable/understand_the_basics/)
- Argo CD, [Core Concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)
- Argo CD, [Architectural Overview](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)
- Argo CD, [Argo CD Core](https://argo-cd.readthedocs.io/en/stable/operator-manual/core/)

---

# Argo CD GitOps Best Practices

## Core idea

GitOps treats reviewed declarative configuration in Git as the desired state of
Kubernetes. Argo CD continuously compares that desired state with the cluster's
live state, reports differences, and reconciles them according to the sync
policy.

This does not remove CI:

- CI builds, tests, scans, signs, and publishes an immutable artifact.
- CI or the release workflow opens a pull request that updates the image digest,
  Chart version, or values in the configuration repository.
- Argo CD pulls the desired state from Git, generates manifests, and continuously
  reconciles Kubernetes.
- Monitoring and verification prove that the new version can serve traffic;
  `Synced` alone is not complete evidence of a successful release.

```text
Application repository                 Configuration repository
source -> CI -> image@sha256:digest -> PR -> review -> merge
                                                    |
                                                    v
Git webhook / polling -> Argo CD -> render -> diff -> sync -> Kubernetes
                                      ^                         |
                                      |                         v
                            desired state              live state + health
```

## Why prefer GitOps-driven CD

| Concern | Pipeline-driven deployment | GitOps-driven deployment |
| --- | --- | --- |
| Execution model | A CI job runs `helm upgrade` or `kubectl apply` once and exits | A controller continuously compares Git with live state |
| Drift | Manual changes can persist unless a separate audit detects them | Drift is reported as `OutOfSync` and can optionally be self-healed |
| Auditability | Pipeline runs, parameters, and logs must be correlated | Git commits, reviews, diffs, and Argo operations form a traceable chain |
| Credentials | The CI runner usually needs cluster write access | CI only updates Git; cluster credentials remain in the CD control plane |
| Recovery | A historical job and its parameters must be reconstructed | Revert a Git commit and let the controller converge |
| Visibility | Job success says little about later state | Sync, operation, and health status remain visible continuously |

GitOps is not better for every operation. One-time administrative actions,
in-database migrations, calls to external SaaS APIs, and incident break-glass
work may still require imperative workflows. The important rule is explicit
ownership: CI, Argo CD, Terraform, and humans using `kubectl` must not write the
same resource or field concurrently.

## Argo CD architecture

```text
Developers / CI / SSO
         |
         v
+-------------------+       webhook       +-------------------+
| Argo CD API Server| <------------------- | Git provider      |
| UI, CLI, API, RBAC|                      +-------------------+
+---------+---------+
          |
          | application operations
          v
+---------+----------+   manifest request  +-------------------+
| Application        | ------------------> | Repository Server |
| Controller          | <------------------ | clone/cache/render|
| compare/reconcile   |    rendered YAML   +-------------------+
+---------+----------+
          |
          | Kubernetes API
          v
+--------------------+
| Managed cluster(s) |
+--------------------+
```

- **API Server** provides the UI, CLI, gRPC/REST API, authentication, RBAC,
  repository and cluster credential management, and webhook handling.
- **Repository Server** clones and caches Git sources and generates manifests
  with Helm, Kustomize, Jsonnet, or plain YAML. Because it handles untrusted
  input, restrict its network access, privileges, and custom plugins.
- **Application Controller** watches `Application` objects and managed
  resources, compares target and live state, updates status, and performs sync,
  hook, and corrective actions.
- **Redis** caches repository, manifest, and application state to reduce Git and
  Kubernetes API load. It is not the source of truth for desired state.
- **ApplicationSet Controller** generates Applications from cluster, Git
  directory, list, and other generators. It fits repeated environments or
  multi-cluster delivery, but a generator or template error can also amplify
  blast radius.

Argo CD Core is a headless installation without the persistent API Server,
OIDC, Argo CD RBAC, or notification controller, and relies mainly on Kubernetes
RBAC. A shared platform that needs SSO, a UI, multi-tenancy, and centralized
governance normally uses the full installation.

## Core resources and status

- `Application` defines the source, revision, path or chart, destination, and
  sync policy for a group of resources.
- `AppProject` limits the source repositories, destination clusters and
  namespaces, and resource kinds available to Applications. It is a
  multi-tenancy and blast-radius boundary.
- `ApplicationSet` uses templates and generators to create Applications. It is
  not the deployment controller for workloads inside an Application.
- **Refresh** fetches target and live state and recalculates differences; it is
  not a sync.
- **Sync status** is `Synced` when live state matches the rendered Git target and
  `OutOfSync` when it differs.
- **Operation status** reports whether the latest sync operation succeeded.
- **Health status** evaluates resource health, such as Deployment availability
  and Pod readiness.

An application can therefore be `Synced` but `Progressing` or `Degraded`. In an
interview or incident, answer "has configuration converged?" separately from
"is the service available?"

## Repository and artifact design

- Use a separate configuration repository when application code and environment
  configuration have different permissions, owners, or release cadences. This
  avoids rebuilding for a replica-only change and creates a cleaner production
  audit history.
- Use immutable references: container image digests, Helm Chart versions, Git
  commit SHAs, and pinned remote Kustomize bases. A floating tag or branch can
  cause the same Git revision to render different manifests over time.
- Do not store plaintext secrets in Git. Use an appropriate mechanism such as
  External Secrets Operator, Secrets Store CSI Driver, or SOPS, and restrict
  decryption privileges and secret exposure.
- Leave room for fields managed by other controllers. When an HPA manages
  replicas, do not also pin `spec.replicas` in Git.
- Do not use Argo CD parameter overrides as a substitute for Git commits. State
  that exists only in the Argo CD API makes Git an incomplete source of truth.

## Application boundaries and ownership

Split Applications by lifecycle, owner, risk, and rollback unit, not only by
namespace or repository:

- Separate application workloads, shared platform controllers, and stateful
  data services so an ordinary application release cannot unexpectedly change
  a database or ingress control plane.
- Give each Kubernetes object one reconciler. First identify ownership by
  Terraform, Helm releases, operator-generated children, CI jobs, and manual
  processes.
- Child resources generated by a controller should normally remain owned by
  that controller instead of being declared in another Argo Application.
- Use AppProject allowlists for repositories, destinations, and resource kinds;
  do not grant every project arbitrary cluster-scoped resources by default.
- A parent app-of-apps or ApplicationSet can create privileged Applications.
  Only platform administrators should be able to modify its source and template.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  sourceRepos:
    - https://git.example.com/platform/payments-config.git
  destinations:
    - namespace: payments
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-production
  namespace: argocd
spec:
  project: payments
  source:
    repoURL: https://git.example.com/platform/payments-config.git
    targetRevision: 4d2c0f6
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    syncOptions:
      - CreateNamespace=false
```

The example deliberately starts without automated sync, prune, or self-heal. A
real production policy should add the RBAC allowlists or denylists, sync windows,
and approval model required by the platform; this example is not a complete
security baseline.

## Safely adopting existing resources

1. **Inventory:** Record live manifests, UIDs, Service ClusterIPs, PVCs,
   credentials, generated children, field managers, health, and every existing
   writer.
2. **Build the matching desired state:** Render the version that is actually
   running. Do not combine adoption with an upgrade, rename, or redesign.
3. **Constrain scope:** Create a least-privileged AppProject and a manual,
   non-pruning Application without self-heal.
4. **Review the diff first:** Separate expected differences, API defaulting,
   mutating webhooks, controller-owned fields, and real drift. Do not hide an
   unknown problem with a broad `ignoreDifferences` rule.
5. **Sync in small units:** Start with one low-risk unit and verify rollout,
   traffic, identity, storage, data, and external side effects, not only
   `Synced`.
6. **Retire the old writer:** Disable the former pipeline deployment, Helm
   automation, or manual procedure so only Argo CD writes the resource.
7. **Add automation progressively:** After observation, evaluate self-heal and
   then prune only for inventoried resources. Protect `allowEmpty` and cascading
   deletion especially carefully.
8. **Exercise recovery:** Test Git revert, failed sync, controller outage,
   credential rotation, Argo CD backup and restore, and reconciliation after a
   break-glass change.

For stateful workloads, also preserve PVC and Service identity, membership,
quorum, backups, and restore proof. A Git revert can restore manifests, but it
cannot automatically reverse an executed database migration or restore lost
data.

## Automated sync, ordering, and release

- `automated` syncs target/live differences; `selfHeal` corrects live drift;
  `prune` deletes tracked resources removed from Git. These are distinct risks
  and require separate decisions.
- Sync phases and resource hooks can implement PreSync migrations, Sync
  workloads, and PostSync verification. Hooks must be idempotent and have
  timeouts, retries, and failure handling.
- Sync waves order resource application, but do not replace dependency
  readiness, backward-compatible schemas, or application-level health checks.
- Use sync windows for freeze periods or production change windows, with a
  designed emergency override and audit trail.
- Prefer Git revert as a new, reviewable desired-state commit. Database schemas,
  PVCs, CRDs, and external side effects need separate, rehearsed recovery paths.

## Security and operations checklist

- [ ] SSO/OIDC, Argo CD RBAC, and Kubernetes RBAC use least privilege and
  separate administrator and application-team responsibilities.
- [ ] AppProjects restrict sources, destinations, cluster-scoped kinds, and
  namespace resources.
- [ ] Repository, cluster, and secret-store credentials are rotatable, absent
  from manifests, and protected from repository-server exposure.
- [ ] Git branch protection, required review, and signed commits, tags, or
  provenance match the supply-chain risk.
- [ ] Images, Charts, Git dependencies, and remote bases use immutable versions.
- [ ] Argo CD, its CRDs, and dependencies have pinned versions, upgrade
  rehearsals, backups, and a disaster-recovery runbook.
- [ ] Monitoring covers controller queues and reconciliation, Git and Kubernetes
  API errors, cache behavior, `OutOfSync`, `Unknown`, `Degraded`, and prolonged
  `Progressing` states.
- [ ] Notifications have an owner and link to the commit, diff, operation, and
  runbook.
- [ ] Break-glass changes are time-bounded and audited, then committed to Git or
  reverted by self-heal.
- [ ] The evidence chain covers commit, artifact digest, rendered manifest, sync
  result, rollout, traffic, SLOs, and data integrity.

## Interview focus

**How do Argo CD and GitLab CI divide responsibilities?**  
CI builds and validates one immutable artifact and updates environment
configuration through a pull request. Argo CD continuously performs CD
reconciliation. CI must stop directly deploying the same Argo-managed resources.

**Does `Synced` mean a deployment succeeded?**  
No. It only proves that live state matches rendered target state. Also verify the
operation, health, rollout, traffic, SLOs, migrations, and external dependencies.

**Why not enable prune and self-heal on day one?**  
An existing cluster may contain uninventoried resources, another writer,
defaulted or mutated fields, or an incorrect Application scope. Manual,
non-pruning adoption proves the diff and ownership before automation can delete
or overwrite at scale.

**How does AppProject differ from Kubernetes RBAC?**  
An AppProject constrains where an Argo Application can deploy from and to, and
which kinds it may manage. Kubernetes RBAC constrains what Argo components and
users can do through the Kubernetes API. Both need least privilege; neither
replaces the other.

## References

- Argo CD, [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- Argo CD, [Understand the Basics](https://argo-cd.readthedocs.io/en/stable/understand_the_basics/)
- Argo CD, [Core Concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)
- Argo CD, [Architectural Overview](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)
- Argo CD, [Argo CD Core](https://argo-cd.readthedocs.io/en/stable/operator-manual/core/)
