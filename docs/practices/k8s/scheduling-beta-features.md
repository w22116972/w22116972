# Kubernetes Beta Scheduling Features 延伸指南

本文件隔離 [Kubernetes Scheduling](scheduling.md) production baseline 以外、在 v1.35-v1.36 仍為 Beta 的 scheduler optimization。

## Opportunistic Batching

Opportunistic Batching 在 v1.35 為 Beta 且預設開啟。Scheduler 可在 processing queue 中找到能重用部分 scheduling work 的 Pods，以 throughput 換取更複雜的 queue behavior。它是 control-plane performance optimization，不改變 Pod 的 requests、constraints 或成功排程保證。

採用或調整前，應以 representative workload 比較 scheduling throughput、p50/p99 latency、unschedulable attempts、queue wait、plugin duration 與 placement quality。若 managed control plane 不提供 scheduler configuration，需接受 provider 的版本與 rollout 邊界，不應假設可自行調 gate。

## 參考資料

- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)

---

# Kubernetes Beta Scheduling Features Supplement

This guide isolates scheduler optimization outside the [Kubernetes Scheduling](scheduling.md) production baseline that remains Beta in v1.35-v1.36.

## Opportunistic Batching

Opportunistic Batching is Beta and enabled by default in v1.35. The scheduler can find Pods in the processing queue that reuse part of the scheduling work, trading additional queue complexity for throughput. It is a control-plane performance optimization; it does not change Pod requests, constraints, or guarantees of successful placement.

Before adoption or tuning, compare scheduling throughput, p50 and p99 latency, unschedulable attempts, queue wait, plugin duration, and placement quality under representative workload. If a managed control plane does not expose scheduler configuration, accept the provider's version and rollout boundary rather than assuming the gate can be adjusted directly.

## References

- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
