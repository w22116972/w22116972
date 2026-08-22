# Kubernetes Policy Preview Features 延伸指南

本文件隔離 [Kubernetes Policy](policy.md) production baseline 以外的 preview capability。

## Local ephemeral storage quota

以 `requests.ephemeral-storage`、`limits.ephemeral-storage` 管理 local ephemeral storage quota 在 v1.36 仍為 Alpha。它與 stable CPU、memory、object-count quota 不同；是否正確 accounting 還受 kubelet storage layout、container logs、writable layers、`emptyDir` 與 filesystem measurement 影響。

採用前要在實際 runtime 與 node filesystem layout 驗證 admission、usage accounting、quota exhaustion、eviction 與 log rotation。不要把 quota acceptance 當成 node disk pressure 已受到保護；仍需 requests/limits、eviction thresholds、monitoring 與容量規劃。

## 參考資料

- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

---

# Kubernetes Policy Preview Features Supplement

This guide isolates preview capability outside the [Kubernetes Policy](policy.md) production baseline.

## Local ephemeral-storage quota

Quota management through `requests.ephemeral-storage` and `limits.ephemeral-storage` remains Alpha in v1.36. It differs from stable CPU, memory, and object-count quota. Correct accounting also depends on kubelet storage layout, container logs, writable layers, `emptyDir`, and filesystem measurement.

Before adoption, test admission, usage accounting, quota exhaustion, eviction, and log rotation against the actual runtime and node filesystem layout. Quota acceptance does not prove that node disk pressure is controlled; requests and limits, eviction thresholds, monitoring, and capacity planning remain necessary.

## References

- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
