# Kubernetes Administration Preview Features 延伸指南

本文件隔離 Kubernetes v1.36 中尚未 GA 的 cluster administration capabilities；production baseline 見 [Kubernetes Cluster Administration](admin.md)。

## Graceful Node Shutdown

Graceful Node Shutdown 在 Linux 為 Beta；Windows support 也仍為 Beta。Linux implementation 依賴 systemd inhibitor locks。即使 feature gate 預設開啟，`shutdownGracePeriod` 與 `shutdownGracePeriodCriticalPods` 預設都是 `0s`，未設定非零期間就不會實際延遲 shutdown。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 60s
shutdownGracePeriodCriticalPods: 20s
```

- 上例先給一般 Pods 40 秒，再給 critical Pods 20 秒；critical period 必須小於總期間。
- Priority-based shutdown ordering 仍為 Beta，必須另外驗證 priority groups、termination time 與 starvation。
- Shutdown 取消後 kubelet 不會還原已開始終止的 Pods；controllers 必須重新建立。
- 驗證 OS、systemd packages 與 cloud termination path 確實觸發 inhibitor lock；不要讓它取代可觀測的 `cordon`/`drain` maintenance workflow。

## Coordinated Leader Election

Coordinated Leader Election 在 v1.36 為 Beta、預設關閉。它讓 mixed-version control-plane replicas 在 upgrade 時依 emulation/binary version 與 creation timestamp 協調 leader preference，降低較新 binary 過早成為 leader 的風險。

它不提供 etcd quorum、network、API endpoint 或 component-state 健康保證。只有在所有 participating components、feature gates、API resources、version-skew plan 與 rollback 都已驗證時才實驗，並監控 election duration、leader churn 與 unavailable intervals。

## 參考資料

- [Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Coordinated Leader Election](https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)

---

# Kubernetes Administration Preview Features Supplement

This guide isolates cluster-administration capabilities that are not GA in Kubernetes v1.36. See [Kubernetes Cluster Administration](admin.md) for the production baseline.

## Graceful Node Shutdown

Graceful Node Shutdown is Beta on Linux, and Windows support also remains Beta. The Linux implementation depends on systemd inhibitor locks. Although the feature gate is enabled by default, `shutdownGracePeriod` and `shutdownGracePeriodCriticalPods` both default to `0s`; shutdown is not delayed without non-zero periods.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 60s
shutdownGracePeriodCriticalPods: 20s
```

- This example gives regular Pods 40 seconds, then critical Pods 20 seconds; the critical period must be shorter than the total.
- Priority-based shutdown ordering remains Beta and requires separate testing of priority groups, termination time, and starvation.
- Kubelet does not restore Pods whose termination started before shutdown was canceled; controllers must recreate them.
- Verify that the OS, systemd packages, and cloud termination path trigger the inhibitor lock. Do not let this replace an observable `cordon` and `drain` maintenance workflow.

## Coordinated Leader Election

Coordinated Leader Election is Beta and disabled by default in v1.36. During a mixed-version control-plane upgrade, it coordinates leader preference using emulation or binary version and creation timestamp, reducing the chance that a newer binary becomes leader too early.

It does not guarantee etcd quorum, network health, API endpoint availability, or component-state correctness. Experiment only after verifying all participating components, feature gates, API resources, version-skew plan, and rollback, while monitoring election duration, leader churn, and unavailable intervals.

## References

- [Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Coordinated Leader Election](https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)
