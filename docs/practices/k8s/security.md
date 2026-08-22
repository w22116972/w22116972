# Kubernetes 安全：面試深度指南

Kubernetes security 不是單一功能，而是跨越 application、container、cluster 與 cloud／infrastructure 的 defense in depth。安全設計必須先辨識 assets、identities、trust boundaries 與可能的 API bypass paths，再用 preventive、detective 與 recovery controls 降低 blast radius。

本文件著重架構與面試推理；實作細節另見 [Pod Security Standards 最佳實務](k8s-pod-security-standards-bp.md)、[Secret 最佳實務](k8s-secure-config-secret-bp.md)、[Networking 指南](networking.md) 與 [PKI 指南](k8s-pki-certificate-bp.md)。

## 先掌握 security control chain

```text
human / workload / controller
              |
              | TLS + credential
              v
        kube-apiserver
              |
              +-- authentication: 你是誰？
              +-- authorization: 你能做什麼？
              +-- admission: 這個 request/object 是否符合 policy？
              +-- validation: object 是否符合 API schema？
              |
              v
             etcd

runtime path: image -> admission -> scheduler -> kubelet -> CRI -> Linux kernel
data path:    client -> gateway/service -> network policy -> Pod -> dependency
```

- Authentication 建立 identity；它不授權動作。
- Authorization 通常由 RBAC 判斷 verb、resource、namespace 與 subresource。啟用多個 authorizers 時，只要其中一個允許，request 就可能通過。
- Admission 在 authorization 後檢查或修改 create、update、delete、connect 等 requests；它通常不攔截純 read。
- Audit 記錄經 API server 的活動，但無法看見所有直接 node、kubelet、etcd 或 static Pod 路徑。
- SecurityContext、seccomp、AppArmor／SELinux 與 user namespaces 在 workload 已被接受後限制 runtime 行為，不能取代 API access control。

面試重點：RBAC 說「caller 可否要求這個操作」，admission 說「送進來的 object 可否存在」，runtime controls 則限制「process 執行後能做什麼」。三者責任不同，不能互相替代。

## 以生命週期與分層設計 defense in depth

| 層級 | 主要風險 | 核心 controls |
| --- | --- | --- |
| Application | insecure code、過度信任內網、Secret 洩漏 | threat modeling、secure coding、input validation、short-lived credentials |
| Supply chain | 惡意或有漏洞的 dependency/image | isolated builds、SBOM、scan、sign、verify、immutable digest、patch process |
| Cluster | 過度 RBAC、policy gap、public control plane | strong authentication、least privilege、admission、audit、encryption at rest |
| Runtime/node | container escape、host access、kernel attack surface | non-root、PSS Restricted、seccomp、LSM、patched minimal OS、node isolation |
| Network/data | unrestricted lateral movement、unencrypted traffic/data | default-deny NetworkPolicy、TLS、KMS、backup protection、egress control |
| Cloud/infrastructure | stolen cloud identity、metadata or management-plane access | workload identity、private endpoints、firewall、cloud IAM、central logging |

Controls 也要涵蓋 develop、distribute、deploy 與 runtime phases。只在 registry 做 image scan，無法阻止部署錯誤 RBAC；只套 PSS，也無法修正 vulnerable application 或被竊取的 cloud credential。

## Authentication 與 human identity

- Production human access 優先整合外部 OIDC／enterprise identity provider，搭配 MFA、短 session、group-based authorization 與集中式 offboarding。
- 只啟用必要的 authentication mechanisms。來源越多，越容易遺漏仍有效的 credential，也越難完整 audit。
- Kubernetes 沒有內建的一般 user database；API server 接收 authenticator 提供的 username 與 groups，再交給 authorizer。
- 不要把長效 X.509 client certificates 當作日常多人登入方案。它們難以逐張撤銷；若仍需使用，縮短 lifetime 並嚴格保護 CA。
- `system:masters` 會繞過 RBAC，僅可作為受控 break-glass identity；記錄取用、告警、定期測試並立即輪替使用過的 credentials。
- 禁用不必要的 anonymous access，限制 API endpoint exposure，並保護 kubeconfig、bootstrap tokens、client certificates 與 identity-provider configuration。

## RBAC：least privilege 與 privilege escalation

優先使用 namespace-scoped `RoleBinding`，避免 wildcard、`cluster-admin` 與不必要的 `ClusterRoleBinding`。授權審查不能只看 role 名稱，必須展開 aggregated roles、groups、bindings 與實際 verbs/resources。

| 權限 | 為何可能升權 |
| --- | --- |
| 建立 Pod／修改 workload | 可掛載該 namespace 的 Secret、使用強大 ServiceAccount，或要求 host access |
| `get/list/watch` Secrets | 三者都可能取得 Secret data；`list/watch` 不是低風險 metadata 權限 |
| 建立 RoleBinding | 可將既有高權限 role 綁給自己或他人，取決於 bind/escalate checks |
| `impersonate` | 可代表其他 user、group 或 ServiceAccount 發送 requests |
| 修改 admission webhook | 可繞過 policy、攔截 credentials，或造成 cluster-wide denial of service |
| 修改 Node／kubelet-related subresources | 可能取得 kubelet API 能力，繞過 admission 與一般 API audit path |
| 建立 privileged workload | 可能控制 node，進而危及同 node workloads 與 node credentials |

實務上要定期執行 access review，移除 stale bindings，對高風險 verbs 與 resources 建立 alert，並用 `kubectl auth can-i --as ...` 驗證 effective permission。不要把 namespace 視為完整安全邊界：能建立 workload 的 principal 通常對該 namespace 已具有很強的間接能力。

## ServiceAccount 與 workload identity

- 每個 workload 使用專用 ServiceAccount；不要共用 `default` ServiceAccount。
- 不需要 Kubernetes API 時設定 `automountServiceAccountToken: false`。
- 需要 API access 時只授予必要 verbs/resources，並使用 TokenRequest／projected、短效、bound ServiceAccount token。
- Token 的 `audience` 應與接收服務匹配；外部服務不應無條件接受給 Kubernetes API 使用的 token。
- 存取 cloud API 時使用 cloud-native workload identity，避免把長效 cloud access key 放進 Kubernetes Secret。
- ServiceAccount 是 namespaced identity，但 ClusterRoleBinding 可讓它取得 cluster-wide 權限；不要把「namespaced」誤解成權限天然受限於 namespace。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-api
  namespace: orders
automountServiceAccountToken: false
```

## Admission、PSS 與 policy rollout

PodSecurityPolicy 已從 Kubernetes v1.25 移除，不應出現在新設計。使用 stable 的 Pod Security Admission（PSA）強制 Pod Security Standards（PSS），或使用經審核的 policy engine 補充 PSS 未涵蓋的規則。

- 一般 application namespace 以 `restricted` 為目標；相容性問題先在 `warn` 與 `audit` 修正，再啟用 `enforce`。
- 將 `enforce-version` 固定在測試過的 Kubernetes minor version，升級前用 `warn-version: latest` 評估差異。
- privileged system components 放在專用 namespace，嚴格限制 writers；例外要有 owner、理由、到期日與 compensating controls。
- Admission webhook 本身是高權限、high-availability dependency。限制修改權限、使用 TLS、設定 timeout/failure policy、監控 latency 與 rejection，避免 fail-open gap 或 fail-closed outage。
- PSA 的 `enforce` 作用在實際 Pod；對 Deployment 等 workload templates 的 audit/warn 可較早暴露問題。

詳細 rollout 與 manifest 範例見 [Kubernetes Pod Security Standards 最佳實務](k8s-pod-security-standards-bp.md)。

## Workload 與 Linux runtime hardening

```yaml
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: registry.example/orders@sha256:<digest>
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          memory: 256Mi
```

- 預設 non-root、禁止 privilege escalation、drop all Linux capabilities、使用 `RuntimeDefault` seccomp，並只加回已驗證的例外。
- 避免 `privileged`、host namespaces、`hostPath`、host ports 與 unsafe sysctls。這些設定會削弱 container boundary 或擴大 node blast radius。
- `readOnlyRootFilesystem` 需要搭配最小化的 writable volume；否則可能破壞 application startup 或 runtime。
- AppArmor／SELinux 能做 mandatory access control；seccomp 過濾 syscalls。選擇與 node OS/runtime 相容的 profiles，並測試 enforcement，而非只寫 manifest。
- User namespaces 可把 container 內的 root 映射成 host 上非 root identity，但支援度與 operational constraints 必須依目標 Kubernetes、runtime、kernel 與 storage driver 驗證；它不是 privileged Pod 的通用豁免。
- RuntimeClass/sandboxed runtime 或專用 nodes 適合 untrusted code、多客戶 workload 或需要更強 kernel boundary 的情境，但會增加 cost、latency 與運維複雜度。
- 若 Linux node 啟用 swap，確認 kernel 與 Kubernetes 配置能防止 memory-backed Secret volume 被寫入 persistent swap；官方建議 kernel 6.3+ 或具 `noswap` backport。

## Node、control plane 與 API bypass paths

API server 是主要 policy enforcement point，但不是唯一能改變 runtime state 的入口：

- 能寫 static Pod manifest path 的使用者可直接讓 kubelet 啟動 workload；該 Pod 甚至可能不完整出現在 API inventory。
- 直接存取 kubelet API 可能避開 admission control 與 Kubernetes audit logging。啟用 kubelet authentication/authorization、驗證 serving certificate，並以 network controls 限制來源。
- 直接修改 etcd 會跳過 authentication、authorization、admission、validation 與 audit；etcd 只允許 control plane 存取，使用 mutual TLS，並保護 backups。
- 能修改 node filesystem、container runtime socket、CNI/CSI plugin 或 privileged DaemonSet 的 identity，通常能跨越 workload isolation。

因此要使用 minimal、patched node OS；限制 SSH/SSM 與 host administration；隔離 control plane、etcd、kubelet 和 workload networks；監控 host filesystem、runtime 與 privileged workloads。API audit 必須搭配 node、cloud、identity-provider 與 runtime telemetry。

## Network、Secret 與 supply-chain controls

- 確認 CNI 真正支援 NetworkPolicy，再為每個 namespace 建立 default-deny ingress/egress，逐一 allow DNS、dependencies 與必要 external destinations。
- NetworkPolicy 通常是 L3/L4 control，不會驗證 application identity 或 payload；需要時搭配 mTLS、service identity、gateway/WAF 與 application authorization。
- API server、etcd、kubelet API 不應公開暴露；control-plane endpoints 使用 private access 或嚴格 allowlist。
- Secret 啟用 encryption at rest，限制 `get/list/watch`，優先掛載到真正需要的 container，並建立 rotation 與 revocation 流程。Base64 不是 encryption。
- Build 使用受控來源與 isolated runner；掃描 dependencies/images、產生 SBOM、簽章與驗證 provenance，部署時固定 immutable digest。
- Image scan 是時間點檢查；必須持續追蹤新 CVE、設定 patch SLA，並能安全 rollback。

Secret 詳細做法見 [Kubernetes 安全設定與 Secret 最佳實務](k8s-secure-config-secret-bp.md)，NetworkPolicy 設計見 [Kubernetes Networking 指南](networking.md)。

## Multi-tenancy：先定義 tenant 與信任程度

Kubernetes 沒有 first-class tenant object。設計前先回答 tenant 是 team、environment、customer 還是 hostile workload，以及它是否能直接存取 API。

| 模型 | 適用情境 | 主要限制 |
| --- | --- | --- |
| Shared namespaces | 高信任、共同 ownership 的 workload | namespace 內橫向升權與 Secret exposure 風險高 |
| Namespace per tenant | 內部 teams、有限互信 | cluster-scoped resources、nodes、webhooks、CRDs 仍共享 |
| Virtual control plane | 需要隔離 API objects，但仍希望共享部分 infrastructure | 額外 components、cost 與 debugging complexity |
| Dedicated cluster/account | hostile tenants、regulatory boundary、不同 administrators | 最高 isolation，也有最高 platform overhead |

Namespace tenancy 至少要組合 RBAC、PSA、ResourceQuota、LimitRange、NetworkPolicy、Secret separation、node placement 與 policy-as-code。對 untrusted tenants，還要評估 dedicated nodes、sandboxed runtime、separate control plane 或 separate cluster；單靠 namespace 不是 hard security boundary。

## Scheduler 與 DRA 的進階 hardening

- Scheduler 是 control-plane security boundary；保護其 kubeconfig，限制 endpoint exposure，停用 production profiling，並避免讓不受信任者修改 scheduler configuration、profiles 或 high-priority placement controls。
- 錯誤的 scheduler policy 可把敏感與不受信任 workloads 放到相同 nodes，或透過 eviction/autoscaling interaction 擴大 denial-of-service。
- Dynamic Resource Allocation（DRA）讓 drivers、scheduler 與 controllers 更新 ResourceClaim 狀態，因此只授予所需 subresources 與 verbs，不要用廣泛 wildcard。
- DRA 的非 GA authorization details 不屬於本 production baseline；需要評估 synthetic subresources 時，參考 [Kubernetes Security Preview Features](security-preview-features.md)。
- Device access 可能等同直接接觸 host hardware、firmware 或 confidential data；將 DRA driver 視為 privileged infrastructure，隔離其 ServiceAccount、nodes 與 release pipeline。

## Security operations 與驗證

### 部署前

- Threat model 已列出 assets、trust boundaries、abuse paths、security owners 與 residual risks。
- Rendered manifests 通過 schema、policy、Secret、image 與 RBAC checks；admission dry run 使用目標 cluster version。
- Image 以 digest pinning，來源與 signature/provenance 可驗證；沒有把 credentials 寫進 image、Git 或 CI logs。
- 每個 workload 有 requests/limits、probes、PSS-compatible SecurityContext、專用 ServiceAccount 與明確 network flows。

### Runtime

- 收集 API audit、authentication、cloud control-plane、node/runtime、admission、DNS/network 與 workload logs，並集中保護 retention 與查詢權限。
- Alert 高風險行為：break-glass 使用、RBAC/admission 變更、Secret 大量讀取、privileged Pod、exec/attach、node shell、anonymous request 與 policy rejection spike。
- 定期 review access、exceptions、certificates、tokens、images、nodes、webhooks 與 cluster add-ons；升級前驗證 PSS 與 feature-status 差異。
- 演練 credential revocation、compromised workload isolation、node replacement、etcd restore 與 admission outage。Backup 成功不代表 restore 可用。

### Incident response

1. 保留 audit、cloud、node 與 runtime evidence，記錄時間範圍與 identity。
2. 先限制 compromised credential、workload、node 與 egress，再評估是否 rotate cluster-wide trust anchors。
3. 沿 identity -> API request -> admission -> scheduler -> node -> network/data path 確認 blast radius。
4. 從 trusted source rebuild/redeploy；不要只刪除可疑 Pod 後宣布恢復。
5. 驗證 effective RBAC、Secret rotation、node integrity、persistence mechanisms 與 downstream systems，再恢復 traffic。

## 面試題與精準回答

### Authentication、authorization 與 admission 有何差異？

Authentication 驗證 caller identity；authorization 判斷該 identity 能否執行 verb/resource；admission 檢查或修改已授權的 request/object。通過 RBAC 不代表 object 符合安全政策，通過 admission 也不代表 runtime process 沒有漏洞。

### 為何能建立 Pod 可能等同取得 Secret？

因為 caller 可建立 Pod 並引用同 namespace 的 Secret，或使用具權限的 ServiceAccount。評估 RBAC 時必須分析間接 capability，而不只是是否直接擁有 `get secrets`。

### PSS Restricted 是否讓 Pod 完全安全？

否。它提供通用 Pod-level hardening baseline，但不處理 vulnerable code、image provenance、RBAC、NetworkPolicy、Secret lifecycle、node compromise 或 application authorization。它是 defense-in-depth 的一層。

### Namespace 是 security boundary 嗎？

它是重要的 administrative 與 policy scope，但不是完整 hard boundary。Nodes、cluster-scoped APIs、CRDs、webhooks 與部分 infrastructure 仍共享；namespace 內能建立 workload 的 principal 也可能取得強大間接能力。

### 為何 API audit 仍可能漏掉攻擊？

Static Pods、direct kubelet API、etcd、node filesystem 與 container runtime paths 可能繞過 API server。要把 API audit 與 host、runtime、cloud、network 和 identity-provider telemetry 關聯。

### Container isolation 等同 VM isolation 嗎？

不等同。一般 containers 共享 host kernel；seccomp、capabilities、LSM、non-root 與 namespaces 能縮小攻擊面，但高風險或 hostile workloads 可能需要 sandboxed runtime、dedicated nodes 或 separate clusters。

## 版本與 feature status

- PodSecurityPolicy 已於 v1.25 移除，本文件不提供其設定方式。
- Pod Security Admission 自 v1.25 為 stable；policy rules 會隨 minor versions 演進，因此 production `enforce-version` 應固定並經測試。
- DRA core security model 保留於本指南；Alpha/Beta authorization details 已移至 [Kubernetes Security Preview Features](security-preview-features.md)。
- User namespaces、kernel features 與 security profiles 的可用性取決於 Kubernetes、OS、kernel、container runtime 與 storage/network integrations；上線前查 compatibility matrix 並實測。

## 參考資料

- [Cloud Native Security and Kubernetes](https://kubernetes.io/docs/concepts/security/cloud-native-security/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [PodSecurityPolicy removal notice](https://kubernetes.io/docs/concepts/security/pod-security-policy/)
- [Security for Linux Nodes](https://kubernetes.io/docs/concepts/security/linux-security/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Good Practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Hardening Guide - Authentication Mechanisms](https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)
- [Hardening Guide - Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)
- [Hardening Guide - Scheduler Configuration](https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/)
- [Kubernetes API Server Bypass Risks](https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/)
- [Linux Kernel Security Constraints for Pods and Containers](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [Application Security Checklist](https://kubernetes.io/docs/concepts/security/application-security-checklist/)

---

# Kubernetes Security: Interview-Depth Guide

Kubernetes security is not a single feature. It is defense in depth across the application, container, cluster, and cloud or infrastructure layers. Start by identifying assets, identities, trust boundaries, and API bypass paths, then combine preventive, detective, and recovery controls to reduce blast radius.

This guide focuses on architecture and interview reasoning. For implementation detail, see [Pod Security Standards Best Practices](k8s-pod-security-standards-bp.md), [Secret Best Practices](k8s-secure-config-secret-bp.md), the [Networking Guide](networking.md), and the [PKI Guide](k8s-pki-certificate-bp.md).

## Start with the security control chain

```text
human / workload / controller
              |
              | TLS + credential
              v
        kube-apiserver
              |
              +-- authentication: who are you?
              +-- authorization: what may you do?
              +-- admission: does this request/object satisfy policy?
              +-- validation: does the object satisfy the API schema?
              |
              v
             etcd

runtime path: image -> admission -> scheduler -> kubelet -> CRI -> Linux kernel
data path:    client -> gateway/service -> network policy -> Pod -> dependency
```

- Authentication establishes identity; it does not grant actions.
- Authorization, commonly RBAC, evaluates verbs, resources, namespaces, and subresources. If multiple authorizers are enabled, an allow from one can be sufficient.
- Admission runs after authorization and can reject or mutate create, update, delete, connect, and similar requests; it generally does not intercept reads.
- Audit covers activity through the API server, not every direct node, kubelet, etcd, or static Pod path.
- SecurityContext, seccomp, AppArmor or SELinux, and user namespaces constrain runtime behavior after admission; they do not replace API access control.

Interview distinction: RBAC answers whether a caller may request an operation, admission answers whether the submitted object may exist, and runtime controls constrain what the process can do after it starts.

## Design defense in depth across layers and lifecycle

| Layer | Primary risks | Core controls |
| --- | --- | --- |
| Application | insecure code, trusted internal networks, Secret leakage | threat modeling, secure coding, input validation, short-lived credentials |
| Supply chain | malicious or vulnerable dependencies and images | isolated builds, SBOM, scanning, signing, verification, immutable digests, patch process |
| Cluster | excessive RBAC, policy gaps, public control plane | strong authentication, least privilege, admission, audit, encryption at rest |
| Runtime/node | container escape, host access, kernel attack surface | non-root, Restricted PSS, seccomp, LSM, patched minimal OS, node isolation |
| Network/data | lateral movement, unencrypted traffic or data | default-deny NetworkPolicy, TLS, KMS, backup protection, egress controls |
| Cloud/infrastructure | stolen cloud identity or management-plane access | workload identity, private endpoints, firewall, cloud IAM, central logging |

Controls must cover develop, distribute, deploy, and runtime phases. Registry scanning cannot prevent bad RBAC, and PSS cannot fix vulnerable code or stolen cloud credentials.

## Authentication and human identity

- For production human access, prefer an external OIDC or enterprise identity provider with MFA, short sessions, group-based authorization, and centralized offboarding.
- Enable as few authentication mechanisms as practical. More sources make stale credentials harder to find and audit.
- Kubernetes has no built-in general user database. The API server consumes usernames and groups from authenticators and passes them to authorizers.
- Do not use long-lived X.509 client certificates as the normal multi-user login mechanism. They are difficult to revoke individually; if required, use short lifetimes and protect the CA.
- `system:masters` bypasses RBAC. Reserve it for a controlled break-glass identity with access logging, alerting, testing, and post-use credential rotation.
- Disable unnecessary anonymous access, restrict API endpoint exposure, and protect kubeconfigs, bootstrap tokens, certificates, and identity-provider configuration.

## RBAC: least privilege and privilege escalation

Prefer namespace-scoped RoleBindings. Avoid wildcards, `cluster-admin`, and unnecessary ClusterRoleBindings. Reviews must expand aggregated roles, groups, bindings, and effective verbs rather than trust role names.

| Permission | Why it can escalate privilege |
| --- | --- |
| Create Pods or modify workloads | Can mount namespace Secrets, use powerful ServiceAccounts, or request host access |
| `get/list/watch` Secrets | All can expose Secret data; list and watch are not harmless metadata access |
| Create RoleBindings | May bind an existing powerful role, subject to bind and escalate checks |
| `impersonate` | Can send requests as another user, group, or ServiceAccount |
| Modify admission webhooks | Can bypass policy, intercept credentials, or cause cluster-wide denial of service |
| Modify Node or kubelet-related subresources | Can gain kubelet capabilities that bypass admission and normal API audit paths |
| Create privileged workloads | Can lead to node control and compromise co-located workloads and credentials |

Review access periodically, remove stale bindings, alert on high-risk verbs and resources, and validate effective permissions with `kubectl auth can-i --as ...`. A principal that can create workloads often has strong indirect capabilities within that namespace.

## ServiceAccounts and workload identity

- Give each workload a dedicated ServiceAccount; do not share the `default` ServiceAccount.
- Set `automountServiceAccountToken: false` when Kubernetes API access is unnecessary.
- When access is required, grant only necessary verbs and resources and use TokenRequest or projected, short-lived, bound tokens.
- Match token `audience` to the receiving service. External services should not blindly accept tokens intended for the Kubernetes API.
- Use cloud-native workload identity for cloud APIs instead of long-lived cloud keys in Kubernetes Secrets.
- A ServiceAccount is a namespaced identity, but a ClusterRoleBinding can grant cluster-wide permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-api
  namespace: orders
automountServiceAccountToken: false
```

## Admission, PSS, and policy rollout

PodSecurityPolicy was removed in Kubernetes v1.25 and does not belong in new designs. Use stable Pod Security Admission (PSA) to enforce Pod Security Standards (PSS), or add a reviewed policy engine for requirements outside PSS.

- Target `restricted` for ordinary application namespaces. Remediate under `warn` and `audit` before enabling `enforce`.
- Pin `enforce-version` to a tested Kubernetes minor version; use warning against newer rules before an upgrade.
- Isolate privileged system components in dedicated namespaces with tightly restricted writers and time-bounded, owned exceptions.
- Treat admission webhooks as privileged, highly available dependencies. Protect their configuration and TLS; define timeout and failure behavior; monitor latency and rejections.
- PSA enforcement applies to actual Pods; audit and warning against workload templates help expose issues earlier.

See [Kubernetes Pod Security Standards Best Practices](k8s-pod-security-standards-bp.md) for rollout and manifest examples.

## Workload and Linux runtime hardening

```yaml
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: registry.example/orders@sha256:<digest>
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          memory: 256Mi
```

- Default to non-root, deny privilege escalation, drop all capabilities, and use `RuntimeDefault` seccomp. Add back only tested exceptions.
- Avoid privileged mode, host namespaces, `hostPath`, host ports, and unsafe sysctls because they weaken the container boundary or enlarge node blast radius.
- Pair `readOnlyRootFilesystem` with narrowly scoped writable volumes needed by the application.
- AppArmor and SELinux provide mandatory access control; seccomp filters syscalls. Test profiles with the actual node OS and runtime.
- User namespaces can map container root to a non-root host identity, but verify support and operational constraints across the target Kubernetes version, runtime, kernel, and storage drivers. They are not a general privileged-Pod exemption.
- Sandboxed runtimes or dedicated nodes can suit untrusted code and multi-customer workloads, with cost, latency, and operational tradeoffs.
- If Linux node swap is enabled, ensure the kernel and Kubernetes configuration prevent memory-backed Secret volumes from reaching persistent swap. Upstream recommends kernel 6.3+ or a `noswap` backport.

## Node, control plane, and API bypass paths

The API server is the primary enforcement point, but not the only route to runtime state:

- Write access to static Pod manifests can make kubelet start workloads directly, potentially without complete API inventory.
- Direct kubelet API access can bypass admission and Kubernetes audit. Require kubelet authentication and authorization, validate serving certificates, and restrict network sources.
- Direct etcd modification bypasses authentication, authorization, admission, validation, and audit. Restrict etcd to the control plane, require mutual TLS, and protect backups.
- Access to the node filesystem, runtime socket, CNI or CSI plugins, or privileged DaemonSets can cross workload boundaries.

Use a minimal, patched node OS; restrict host administration; segment control-plane, etcd, kubelet, and workload networks; and monitor host filesystems, runtimes, and privileged workloads. Correlate API audit with node, cloud, identity-provider, and runtime telemetry.

## Network, Secret, and supply-chain controls

- Verify that the CNI enforces NetworkPolicy. Apply default-deny ingress and egress per namespace, then allow DNS, dependencies, and required external destinations explicitly.
- NetworkPolicy is generally an L3/L4 control and does not verify application identity or payload. Add mTLS, service identity, gateway/WAF, and application authorization when needed.
- Do not expose the API server, etcd, or kubelet API publicly; use private access or strict allowlists.
- Encrypt Secrets at rest, restrict `get/list/watch`, mount only where required, and define rotation and revocation. Base64 is not encryption.
- Build from controlled sources on isolated runners; scan dependencies and images, produce an SBOM, verify signatures and provenance, and deploy immutable digests.
- Scanning is a point-in-time check. Track newly disclosed vulnerabilities, define patch SLAs, and maintain safe rollback.

See [Kubernetes Secure Configuration and Secret Best Practices](k8s-secure-config-secret-bp.md) and the [Kubernetes Networking Guide](networking.md).

## Multi-tenancy: define the tenant and trust level first

Kubernetes has no first-class tenant object. Decide whether a tenant is a team, environment, customer, or hostile workload, and whether it has direct API access.

| Model | Suitable for | Main limitation |
| --- | --- | --- |
| Shared namespaces | highly trusted workloads with common ownership | high risk of lateral privilege and Secret exposure within the namespace |
| Namespace per tenant | internal teams with limited mutual trust | cluster-scoped resources, nodes, webhooks, and CRDs remain shared |
| Virtual control plane | isolated API objects while sharing some infrastructure | more components, cost, and debugging complexity |
| Dedicated cluster/account | hostile tenants, regulatory boundaries, separate administrators | strongest isolation and highest platform overhead |

Namespace tenancy needs RBAC, PSA, ResourceQuota, LimitRange, NetworkPolicy, Secret separation, node placement, and policy as code. For untrusted tenants, evaluate dedicated nodes, a sandboxed runtime, a separate control plane, or a separate cluster. Namespace alone is not a hard security boundary.

## Advanced scheduler and DRA hardening

- Treat the scheduler as a control-plane security boundary. Protect its kubeconfig, restrict endpoint exposure, disable production profiling, and prevent untrusted changes to scheduler configuration, profiles, or high-priority placement controls.
- A bad scheduler policy can co-locate sensitive and untrusted workloads or amplify denial of service through eviction and autoscaling interactions.
- Dynamic Resource Allocation (DRA) lets drivers, schedulers, and controllers update ResourceClaim status. Grant only the required subresources and verbs, never broad wildcards.
- Non-GA DRA authorization details are outside this production baseline. See [Kubernetes Security Preview Features](security-preview-features.md) when evaluating synthetic subresources.
- Device access can expose host hardware, firmware, or confidential data. Treat DRA drivers as privileged infrastructure and isolate their ServiceAccounts, nodes, and release pipelines.

## Security operations and validation

### Before deployment

- The threat model identifies assets, trust boundaries, abuse paths, owners, and residual risks.
- Rendered manifests pass schema, policy, Secret, image, and RBAC checks; admission dry runs use the target cluster version.
- Images are digest-pinned and have verifiable origin and provenance; credentials are absent from images, Git, and CI logs.
- Every workload has resources, probes, a PSS-compatible SecurityContext, a dedicated ServiceAccount, and explicit network flows.

### Runtime

- Collect and protect API audit, authentication, cloud control-plane, node/runtime, admission, DNS/network, and workload logs.
- Alert on break-glass use, RBAC or admission changes, bulk Secret reads, privileged Pods, exec/attach, node shells, anonymous requests, and policy rejection spikes.
- Periodically review access, exceptions, certificates, tokens, images, nodes, webhooks, and add-ons. Evaluate PSS and feature-status changes before upgrades.
- Exercise credential revocation, compromised-workload isolation, node replacement, etcd restore, and admission outage. A completed backup does not prove usable restore.

### Incident response

1. Preserve API audit, cloud, node, and runtime evidence with time ranges and identities.
2. Constrain compromised credentials, workloads, nodes, and egress before deciding whether trust anchors require rotation.
3. Trace identity -> API request -> admission -> scheduler -> node -> network/data path to establish blast radius.
4. Rebuild and redeploy from trusted sources; deleting a suspicious Pod is not recovery.
5. Verify effective RBAC, Secret rotation, node integrity, persistence, and downstream systems before restoring traffic.

## Interview questions and precise answers

### How do authentication, authorization, and admission differ?

Authentication verifies caller identity. Authorization decides whether that identity may perform a verb on a resource. Admission evaluates or mutates the authorized request and object. Passing RBAC does not prove policy compliance, and passing admission does not prove the runtime process is vulnerability-free.

### Why can permission to create Pods imply access to Secrets?

The caller can create a Pod that references a Secret in the namespace or uses a powerful ServiceAccount. RBAC analysis must evaluate indirect capabilities, not only direct `get secrets` permission.

### Does Restricted PSS fully secure a Pod?

No. It provides a general Pod-hardening baseline but does not cover vulnerable code, image provenance, RBAC, NetworkPolicy, Secret lifecycle, node compromise, or application authorization.

### Is a namespace a security boundary?

It is an important administrative and policy scope, but not a complete hard boundary. Nodes, cluster-scoped APIs, CRDs, and webhooks remain shared, and workload-creation permission can confer strong indirect capabilities.

### Why can API audit miss an attack?

Static Pods, direct kubelet or etcd access, node filesystems, and container runtime paths can bypass the API server. Correlate API audit with host, runtime, cloud, network, and identity-provider telemetry.

### Is container isolation equivalent to VM isolation?

No. Ordinary containers share the host kernel. Non-root, capabilities, seccomp, LSM, and namespaces reduce attack surface, but hostile workloads may require a sandboxed runtime, dedicated nodes, or separate clusters.

## Version and feature status

- PodSecurityPolicy was removed in v1.25; this guide provides no configuration guidance for it.
- Pod Security Admission has been stable since v1.25. Policy rules evolve with minor versions, so pin and test production `enforce-version`.
- This guide retains the core DRA security model; alpha and beta authorization details have moved to [Kubernetes Security Preview Features](security-preview-features.md).
- User namespaces, kernel features, and security profiles depend on Kubernetes, OS, kernel, container runtime, and storage/network integrations. Check compatibility and test before production.

## References

- [Cloud Native Security and Kubernetes](https://kubernetes.io/docs/concepts/security/cloud-native-security/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [PodSecurityPolicy removal notice](https://kubernetes.io/docs/concepts/security/pod-security-policy/)
- [Security for Linux Nodes](https://kubernetes.io/docs/concepts/security/linux-security/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Good Practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Hardening Guide - Authentication Mechanisms](https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)
- [Hardening Guide - Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)
- [Hardening Guide - Scheduler Configuration](https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/)
- [Kubernetes API Server Bypass Risks](https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/)
- [Linux Kernel Security Constraints for Pods and Containers](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [Application Security Checklist](https://kubernetes.io/docs/concepts/security/application-security-checklist/)
