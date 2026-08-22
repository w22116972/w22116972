# Kubernetes Security Preview Features 延伸指南

本文件隔離 [Kubernetes Security](security.md) production baseline 以外、在 v1.36 仍未 GA 的 security capability。

## DRA synthetic subresource authorization

DRA 的 synthetic subresource authorization 在 v1.36 為 Beta 且預設開啟，用來把 `ResourceClaim` 與 `ResourceClaimTemplate` 的部分操作縮小到特定授權路徑。它改善 least privilege，但不能假設 older clusters、provider 或所有 clients 都有相同語意。

- 在 upgrade 前列出實際 API discovery 與 RBAC evaluation 結果，不只檢查 manifest。
- 分離 tenant、DRA driver 與 cluster-admin 權限；不要以 wildcard 補足版本差異。
- 測試 claim create/update/status、driver reconciliation、audit record 與 downgrade/rollback。
- `adminAccess` 仍是 privileged maintenance path，應限制 namespace、principal 與時間範圍。

## 採用 gate

只有在實際 cluster version、provider support、driver compatibility、RBAC tests 與 rollback path 都已驗證時，才依賴此 preview authorization model。否則採用 stable API 可表達的較保守 least-privilege baseline。

## 參考資料

- [DRA Hardening Guide](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)

---

# Kubernetes Security Preview Features Supplement

This guide isolates security capabilities outside the [Kubernetes Security](security.md) production baseline that are still not GA in v1.36.

## DRA synthetic subresource authorization

DRA synthetic subresource authorization is Beta and enabled by default in v1.36. It narrows selected `ResourceClaim` and `ResourceClaimTemplate` operations to specific authorization paths. It improves least privilege, but older clusters, providers, and clients must not be assumed to have identical semantics.

- Inspect actual API discovery and RBAC evaluation results before upgrades, not only manifests.
- Separate tenant, DRA-driver, and cluster-admin permissions; do not use wildcards to hide version differences.
- Test claim create, update, status, driver reconciliation, audit records, and downgrade or rollback.
- `adminAccess` remains a privileged maintenance path and should be constrained by namespace, principal, and time.

## Adoption gate

Depend on this preview authorization model only after verifying the actual cluster version, provider support, driver compatibility, RBAC tests, and rollback path. Otherwise, use a conservative least-privilege baseline expressible through stable APIs.

## References

- [DRA Hardening Guide](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)
