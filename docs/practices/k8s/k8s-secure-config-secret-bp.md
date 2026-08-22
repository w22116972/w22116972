# Kubernetes 安全設定與 Secret 最佳實務

本文件說明如何安全管理 Kubernetes 設定、Secret 與相關 RBAC 權限。Secret 的 Base64 編碼**不是加密**；若未另外設定，API server 會將 Secret 以未加密形式儲存在 etcd。

## 核心原則

- 不要將真實密碼、Token、私鑰、Base64 後的 Secret、解密金鑰或 Helm 渲染結果提交到 Git。
- 一般 manifest、Secret 結構範本及加密過的 Secret 可以納入版本控制，但解密金鑰必須分開保護。
- 不要在 Dockerfile 使用 `ENV` 或 `ARG` 寫入 Secret，也不要將 Secret 放進容器映像。
- 正式環境優先使用外部 Secret 管理服務，並透過 Secrets Store CSI Driver 或經審核的同步控制器提供給工作負載。
- 每個 Secret 只能有一個管理來源。不要讓 Helm、GitOps 與 CI/CD 同時修改同一個 Secret。
- 使用短效憑證，並建立輪替、撤銷及事故處理程序。
- 應用程式不得把 Secret 寫入 log、錯誤訊息、trace、metrics 或傳送給不受信任的服務。

## 一般 Kubernetes 設定

- 使用叢集支援的最新穩定 API；可用 `kubectl api-resources` 確認。
- 將非機密設定存放於版本控制，並透過 review、GitOps 或 CI/CD 套用；不要從個人電腦直接套用未追蹤的正式環境設定。
- 使用 YAML。布林值只使用 `true` 或 `false`；看起來像布林值但實際是字串的內容應加引號。
- 保持 manifest 簡單，避免重複指定 Kubernetes 已管理的預設值。
- 使用共同 labels，並以 `kubernetes.io/description` annotation 說明資源用途；annotation 內不可包含 Secret。
- 正式工作負載應由 Deployment、StatefulSet、DaemonSet 或 Job 管理，不要使用 naked Pod。
- CI/CD 映像應固定在測試過的版本，敏感流程最好再固定 digest，避免使用 `latest`。

## 將 Secret 提供給容器

### 建議：掛載成唯讀檔案

若應用程式支援從檔案讀取憑證，優先使用 Secret volume。Secret volume 為唯讀且由 `tmpfs` 支援。只將 Secret 掛載至需要它的容器。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xxx
  labels:
    app.kubernetes.io/name: xxx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: xxx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: xxx
    spec:
      automountServiceAccountToken: false
      containers:
        - name: xxx
          image: example.invalid/xxx:1.0.0
          volumeMounts:
            - name: credentials
              mountPath: /var/run/secrets/xxx
              readOnly: true
      volumes:
        - name: credentials
          secret:
            secretName: xxx-credentials
            defaultMode: 0400
            items:
              - key: xxx_password
                path: password
```

使用 `items` 時只會掛載列出的 keys，而且每個列出的非 optional key 都必須存在，否則 volume 不會建立。YAML 可使用 `0400` 八進位權限；JSON 不支援八進位 literal，必須改用對應的十進位數值。

Secret volume 的更新採最終一致性，應用程式仍需重新讀取檔案。不要透過 `subPath` 掛載需要輪替的 Secret，因為 `subPath` 不會收到自動更新。

### 必要時才使用環境變數

環境變數可能透過 crash dump、log 或 Linux process environment 洩漏。Secret 更新後，既有容器的環境變數也不會改變；輪替後必須重新建立 Pod。
優先以個別 `secretKeyRef` 明確選取需要的 key；避免用 `envFrom` 將 Secret 的所有 keys 暴露給容器。

```yaml
env:
  - name: XXX_SECRET
    valueFrom:
      secretKeyRef:
        name: xxx-credentials
        key: xxx_password
```

Secret key 必須與 `secretKeyRef.key` 完全一致；非 optional key 不存在時，Pod 無法正常啟動。

## Secret 建立與管理模式

### 正式環境：外部 Secret 管理服務

優先將機密資料保留在雲端 KMS／Secret Manager、Vault 或其他受管理的 Secret store，並讓經授權的 Pod 透過 Secrets Store CSI Driver 掛載。應確認 provider 的輪替、同步、失效處理及稽核行為。

### 簡單環境：由 GitLab CI/CD 管理

只有在未使用外部 Secret store 或 GitOps Secret controller 時，才讓 CI/CD 管理 Secret。下例假設 `XXX_PASSWORD_FILE` 是 GitLab 中受保護的 file-type variable：

```yaml
deploy-secret:
  stage: deploy
  # 請替換成經測試的版本；正式環境建議再固定 digest。
  image: bitnami/kubectl:<tested-version>
  resource_group: production-secret
  script:
    - |
      kubectl create secret generic xxx-credentials \
        --from-file=xxx_password="$XXX_PASSWORD_FILE" \
        --namespace="$KUBE_NAMESPACE" \
        --dry-run=client \
        --output=yaml \
      | kubectl apply --filename=-
    # 只有透過環境變數讀取 Secret 時需要重啟。
    - kubectl rollout restart deployment/xxx --namespace="$KUBE_NAMESPACE"
    - kubectl rollout status deployment/xxx --namespace="$KUBE_NAMESPACE"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

- GitLab 預設檔名是 `.gitlab-ci.yml`。
- 敏感 variable 必須設為 protected 及 masked；file-type variable 或外部 provider 可降低被 `env`／`printenv` 意外輸出的風險。
- Masking 無法阻止惡意 CI script 讀取 Secret，因此 pipeline 變更必須先經 review。
- CI 的 Kubernetes 身分只能取得目標 namespace 內所需的最小權限，不可使用 `cluster-admin` 或 wildcard。
- `create` Secret 權限不能以 `resourceNames` 限制到單一名稱。需要更嚴格隔離時，由平台預先建立 Secret，再讓 CI 只更新指定資源，或改用專用控制器。

### 不建議：以 Helm values 傳遞真實 Secret

不要把真實 Secret 放入已提交的 `values.yaml`、命令列 `--set`、CI log 或 Helm 渲染產物。Helm 應只引用由外部 Secret store、GitOps controller 或 CI/CD 建立的 Secret。
也不要在互動式 shell 使用 `kubectl create secret --from-literal=<secret>`；實際值可能留在 shell history、process list 或終端記錄。優先從受保護檔案或外部 provider 讀取。

若範例或測試環境必須使用 Secret manifest，key 必須一致，且不得提交真實值：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: xxx-credentials
  annotations:
    kubernetes.io/description: Credentials consumed by the xxx workload
type: Opaque
data:
  # Base64 不是加密；不要提交真實內容。
  xxx_password: <BASE64_VALUE>
```

`stringData` 可接受未編碼字串，但不會提高安全性，而且不適合搭配 server-side apply。

## 叢集管理者檢查清單

- 為 Secret 啟用 API data encryption at rest。整合外部金鑰服務時優先使用 KMS v2；KMS v1 自 Kubernetes 1.28 起 deprecated，KMS v2 自 1.29 起 stable。
- 保護 EncryptionConfiguration、KMS 憑證、etcd、etcd 備份與退役磁碟；etcd 成員之間使用 TLS。
- 使用最小權限 RBAC，優先使用 namespace 範圍的 RoleBinding，避免 ClusterRoleBinding、wildcard、`cluster-admin` 與 `system:masters`。
- 嚴格限制 Secret 的 `get`、`list` 及 `watch`；`list` 和 `watch` 也會揭露 Secret 內容。
- 將「可在 namespace 內建立 Pod、Deployment 等工作負載」視為可能讀取該 namespace 中 Secret 的權限。
- 使用 namespace 隔離不同信任層級，但不要把同一 namespace 視為強安全邊界。
- 定期檢查 RBAC binding，移除過期帳號、重複授權及 `system:unauthenticated` 的非必要 binding。
- 對異常或大量 Secret 讀取建立 audit policy 與警示。
- 限制 privileged Pod；privileged container 可能讀取同一節點上其他 Pod 使用的 Secret。

## 應用程式開發者檢查清單

- 每個 workload 使用專用 ServiceAccount，不要依賴 `default` ServiceAccount。
- 不需要 Kubernetes API 時設定 `automountServiceAccountToken: false`。
- 需要 Token 時使用 TokenRequest API 或 projected、短效且自動輪替的 Token，不要建立 legacy 長效 token Secret。
- 多容器 Pod 中，只向真正需要的容器提供 Secret。
- 測試 Secret 輪替，確認應用程式能重新讀取檔案或 Deployment 能安全 rolling restart。
- `immutable: true` 只適合以新名稱建立新版本、不做原地輪替的 Secret。immutable Secret 無法改回 mutable，內容也不能直接更新。

## DRA 範圍說明

Kubernetes 文件中的 DRA 是 **Dynamic Resource Allocation**，用來配置 GPU、加速器或其他裝置，並不是 data-at-rest encryption，也不會提升 Secret 的保密性。

若工作負載有使用 DRA：

- 將 ResourceClaim／ResourceClaimTemplate 與 DRA driver 的管理權限和 Secret 管理權限分開授予。
- 自訂 node drain 流程時，先移除使用 claim 的 Pod，最後才 drain DRA driver，讓 driver 完成裝置清理。
- 大量使用 DRA 時，監控 API server、`kube-controller-manager`、scheduler、kubelet 與 DRA driver 的負載及操作延遲。

## 參考資料

- [Kubernetes Configuration Good Practices](https://kubernetes.io/blog/2025/11/25/configuration-good-practices/)
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [GitLab CI/CD variables](https://docs.gitlab.com/ci/variables/)
- [GitLab deprecated CI/CD keywords](https://docs.gitlab.com/ci/yaml/deprecated_keywords/)

---

# Kubernetes Secure Configuration and Secret Good Practices

This guide explains how to manage Kubernetes configuration, Secrets, and related RBAC permissions securely. Base64 encoding is **not encryption**. Unless encryption at rest is configured separately, the API server stores Secrets unencrypted in etcd.

## Core principles

- Never commit real passwords, tokens, private keys, Base64-encoded Secrets, decryption keys, or rendered Helm output to Git.
- General manifests, Secret templates, and encrypted Secret manifests can be version-controlled, but decryption keys must be protected separately.
- Do not place Secrets in Dockerfile `ENV` or `ARG` instructions or bake them into images.
- For production, prefer an external secret manager and the Secrets Store CSI Driver or a reviewed synchronization controller.
- Give each Secret one management owner. Do not let Helm, GitOps, and CI/CD modify the same Secret.
- Use short-lived credentials and define rotation, revocation, and incident-response procedures.
- Applications must not write Secrets to logs, errors, traces, metrics, or untrusted services.

## General Kubernetes configuration

- Use the latest stable API supported by the cluster; check with `kubectl api-resources`.
- Store non-secret configuration in version control and apply it through review, GitOps, or CI/CD. Do not apply untracked production configuration from a personal workstation.
- Prefer YAML. Use only `true` and `false` for booleans, and quote values that resemble booleans but are strings.
- Keep manifests simple and avoid repeating defaults already managed by Kubernetes.
- Use common labels and a `kubernetes.io/description` annotation; never put Secret data in annotations.
- Production workloads should be managed by a Deployment, StatefulSet, DaemonSet, or Job; avoid naked Pods.
- Pin CI/CD images to a tested version and preferably a digest for sensitive workflows; avoid `latest`.

## Providing a Secret to a container

### Recommended: mount a read-only file

If the application can read credentials from files, prefer a Secret volume. Secret volumes are read-only and backed by `tmpfs`. Mount the Secret only into the container that needs it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xxx
  labels:
    app.kubernetes.io/name: xxx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: xxx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: xxx
    spec:
      automountServiceAccountToken: false
      containers:
        - name: xxx
          image: example.invalid/xxx:1.0.0
          volumeMounts:
            - name: credentials
              mountPath: /var/run/secrets/xxx
              readOnly: true
      volumes:
        - name: credentials
          secret:
            secretName: xxx-credentials
            defaultMode: 0400
            items:
              - key: xxx_password
                path: password
```

When `items` is present, only the listed keys are mounted, and every listed non-optional key must exist or the volume is not created. YAML accepts octal `0400`; JSON does not support octal literals and requires the corresponding decimal value.

Secret volume updates are eventually consistent, and the application must reload the file. Do not use `subPath` for a Secret that needs rotation because it does not receive automated updates.

### Use environment variables only when required

Environment variables can leak through crash dumps, logs, or the Linux process environment. Existing containers do not receive a new value when a Secret changes, so recreate Pods after rotation.
Prefer individual `secretKeyRef` entries that select only required keys; avoid exposing every key from a Secret through `envFrom`.

```yaml
env:
  - name: XXX_SECRET
    valueFrom:
      secretKeyRef:
        name: xxx-credentials
        key: xxx_password
```

The Secret key must exactly match `secretKeyRef.key`. A Pod cannot start normally when a referenced non-optional key is missing.

## Secret creation and ownership models

### Production: external secret manager

Prefer keeping confidential data in a cloud KMS/Secret Manager, Vault, or another managed secret store. Let authorized Pods mount it through the Secrets Store CSI Driver. Verify rotation, synchronization, failure handling, and audit behavior for the provider.

### Simple environments: GitLab CI/CD ownership

Let CI/CD manage the Secret only when there is no external secret store or GitOps controller. This example assumes `XXX_PASSWORD_FILE` is a protected, file-type GitLab variable:

```yaml
deploy-secret:
  stage: deploy
  # Replace with a tested version; production should preferably pin the digest too.
  image: bitnami/kubectl:<tested-version>
  resource_group: production-secret
  script:
    - |
      kubectl create secret generic xxx-credentials \
        --from-file=xxx_password="$XXX_PASSWORD_FILE" \
        --namespace="$KUBE_NAMESPACE" \
        --dry-run=client \
        --output=yaml \
      | kubectl apply --filename=-
    # Restart only when the workload consumes the Secret through an environment variable.
    - kubectl rollout restart deployment/xxx --namespace="$KUBE_NAMESPACE"
    - kubectl rollout status deployment/xxx --namespace="$KUBE_NAMESPACE"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

- The default GitLab filename is `.gitlab-ci.yml`.
- Sensitive variables must be protected and masked. File-type variables or an external provider reduce accidental output by `env` or `printenv`.
- Masking cannot stop malicious CI scripts from reading a Secret; review pipeline changes first.
- Give the CI identity only the required permissions in the target namespace. Do not use `cluster-admin` or wildcards.
- Permission to `create` Secrets cannot be limited to one name using `resourceNames`. For stricter isolation, pre-create the Secret and let CI update only that resource, or use a dedicated controller.

### Not recommended: real Secrets in Helm values

Do not put real Secrets in committed `values.yaml`, command-line `--set`, CI logs, or rendered Helm output. Helm should reference a Secret created by an external secret store, GitOps controller, or CI/CD.
Do not pass a real value through `kubectl create secret --from-literal=<secret>` in an interactive shell either; it can remain in shell history, process listings, or terminal records. Prefer a protected file or external provider.

For examples or tests, keep key names consistent and never commit a real value:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: xxx-credentials
  annotations:
    kubernetes.io/description: Credentials consumed by the xxx workload
type: Opaque
data:
  # Base64 is not encryption; do not commit real content.
  xxx_password: <BASE64_VALUE>
```

`stringData` accepts unencoded strings, but does not improve security and does not work well with server-side apply.

## Cluster administrator checklist

- Enable API data encryption at rest. Prefer KMS v2 with an external key service. KMS v1 has been deprecated since Kubernetes 1.28; KMS v2 has been stable since 1.29.
- Protect EncryptionConfiguration, KMS credentials, etcd, backups, and retired storage. Use TLS between etcd members.
- Apply least-privilege RBAC. Prefer namespace-scoped RoleBindings and avoid ClusterRoleBindings, wildcards, `cluster-admin`, and `system:masters`.
- Strictly limit Secret `get`, `list`, and `watch`; both `list` and `watch` expose Secret contents.
- Treat permission to create workloads in a namespace as potential access to its Secrets.
- Use namespaces to isolate trust levels, but do not treat a namespace as a strong security boundary.
- Periodically review RBAC bindings and remove stale accounts, duplicate grants, and unnecessary `system:unauthenticated` bindings.
- Alert on unusual or bulk Secret reads through audit policy.
- Restrict privileged Pods; a privileged container may access Secrets used by other Pods on its node.

## Application developer checklist

- Use a dedicated ServiceAccount for each workload instead of the `default` ServiceAccount.
- Set `automountServiceAccountToken: false` when Kubernetes API access is not required.
- When a token is required, use TokenRequest or a projected, short-lived, rotating token; avoid legacy long-lived token Secrets.
- In multi-container Pods, expose the Secret only to containers that need it.
- Test rotation and verify that applications reload files or Deployments rolling-restart safely.
- Use `immutable: true` only for versioned Secrets replaced under a new name. Immutable data cannot be updated or made mutable again.

## DRA scope clarification

In Kubernetes, DRA means **Dynamic Resource Allocation** for GPUs, accelerators, and other devices. It is not data-at-rest encryption and does not improve Secret confidentiality.

If a workload uses DRA:

- Grant ResourceClaim/ResourceClaimTemplate and DRA driver administration separately from Secret administration.
- In a custom node-drain workflow, remove Pods with claims first and drain the DRA driver last so it can finish cleanup.
- At scale, monitor API server, `kube-controller-manager`, scheduler, kubelet, and DRA driver load and latency.

## References

- [Kubernetes Configuration Good Practices](https://kubernetes.io/blog/2025/11/25/configuration-good-practices/)
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [GitLab CI/CD variables](https://docs.gitlab.com/ci/variables/)
- [GitLab deprecated CI/CD keywords](https://docs.gitlab.com/ci/yaml/deprecated_keywords/)
