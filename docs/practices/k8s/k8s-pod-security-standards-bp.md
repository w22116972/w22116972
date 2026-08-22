# Kubernetes Pod Security Standards 最佳實務

PodSecurityPolicy 已在 Kubernetes v1.25 移除。新叢集應使用內建且自 v1.25 stable 的 Pod Security Admission（PSA），或經審核的第三方 admission controller，強制執行 Pod Security Standards（PSS）。

## 選擇安全層級

- `privileged` 幾乎不限制 Pod，只適合受嚴格管控的系統工作負載。
- `baseline` 阻擋已知的 privilege escalation，同時保留較高相容性。
- `restricted` 採目前的 Pod hardening 最佳實務，應作為一般應用程式的目標。
- 依 namespace 的信任層級分類，不要因單一例外而降低所有工作負載的政策。
- 將必須 privileged 的 controllers、CNI 或 storage drivers 放在專用 namespace，限制寫入權限並記錄例外原因與 owner。

## 分階段導入 PSA

先使用 `warn` 與 `audit` 觀察違規，再修正 workload 並啟用 `enforce`。不要直接在未盤點的 production namespace 套用 `restricted`。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

- 將 `enforce-version` 固定為叢集支援且已驗證的 minor version，避免升級時無預警改變 admission 結果。
- `warn-version` 與 `audit-version` 可使用 `latest` 提前發現新規則，但必須監控 warning 與 audit events。
- 新 namespace 建立時自動套用最低標準，並持續偵測未標記 namespace。
- Kubernetes system namespaces 及第三方 controllers 需要個別評估；不要盲目批次修改。

## 讓工作負載符合 restricted

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

- 使用 non-root image，避免 privileged、hostPID、hostIPC、hostNetwork、hostPath 與不必要的 capabilities。
- 設定 `seccompProfile.type: RuntimeDefault`，並移除所有 capabilities；只有經驗證的需求才加回允許項目。
- 設定唯讀 root filesystem 時，另外提供應用程式需要的 writable volumes。
- PSS 不處理所有風險；仍需搭配 RBAC、NetworkPolicy、image policy、Secret 管理、runtime hardening 與 node isolation。

## 驗證與維運

- 在 CI 使用與目標叢集相同版本的 policy 驗證 rendered manifests。
- 先以 server-side dry run 或 staging namespace 驗證，再逐步發布 labels。
- 監控 rejected requests、warnings、audit annotations 與 privileged namespace 的寫入者。
- Kubernetes 升級前比較新 PSS 版本，在 `warn`／`audit` 清零後再提升 `enforce-version`。
- 例外必須包含理由、owner、到期日、補償控制與移除計畫。

## 參考資料

- [Enforcing Pod Security Standards](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [PodSecurityPolicy removal](https://kubernetes.io/docs/concepts/security/pod-security-policy/)

---

# Kubernetes Pod Security Standards Best Practices

PodSecurityPolicy was removed in Kubernetes v1.25. New clusters should use the built-in Pod Security Admission (PSA), stable since v1.25, or a reviewed third-party admission controller to enforce Pod Security Standards (PSS).

## Select a security level

- `privileged` is nearly unrestricted and belongs only in tightly controlled system namespaces.
- `baseline` blocks known privilege escalations while retaining broad compatibility.
- `restricted` follows current Pod hardening practices and should be the target for ordinary applications.
- Classify namespaces by trust level. Do not weaken every workload because one component needs an exception.
- Isolate controllers, CNI components, or storage drivers that require privilege; restrict namespace writers and record each exception's reason and owner.

## Roll out PSA in stages

Begin with `warn` and `audit`, remediate workloads, and then enable `enforce`. Do not apply `restricted` directly to an unassessed production namespace.

Use the namespace manifest above as a starting point:

- Pin `enforce-version` to a tested minor version supported by the cluster.
- `warn-version` and `audit-version` can use `latest` to expose future requirements, provided warnings and audit events are monitored.
- Apply a minimum policy automatically to new namespaces and detect unlabeled namespaces continuously.
- Assess system namespaces and third-party controllers individually instead of changing them in bulk.

## Make workloads meet restricted

Use the security context above as a baseline:

- Use non-root images and avoid privileged mode, host namespaces, `hostPath`, and unnecessary capabilities.
- Set `seccompProfile.type: RuntimeDefault`, drop all capabilities, and add back only validated exceptions.
- If the root filesystem is read-only, provide narrowly scoped writable volumes required by the application.
- PSS is not a complete security boundary. Combine it with RBAC, NetworkPolicy, image policy, Secret management, runtime hardening, and node isolation.

## Validate and operate

- Validate rendered manifests in CI using the same policy version as the target cluster.
- Use server-side dry run or a staging namespace before progressively applying labels.
- Monitor rejected requests, warnings, audit annotations, and writers to privileged namespaces.
- Before upgrades, evaluate the new PSS version and raise `enforce-version` only after `warn` and `audit` findings are resolved.
- Every exception needs a reason, owner, expiry, compensating controls, and removal plan.

## References

- [Enforcing Pod Security Standards](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [PodSecurityPolicy removal](https://kubernetes.io/docs/concepts/security/pod-security-policy/)
