# Kubernetes Observability：Metrics、Logs 與 Traces 面試指南

Observability 不是安裝一套 dashboard，而是從 metrics、logs 與 traces 推論系統內部狀態，並把 signal 與 Kubernetes identity、change event 及 service objective 關聯。每一種 signal 回答不同問題；只收集而沒有 retention、cardinality、alert ownership 與 incident workflow，仍不構成可運作的 observability system。

## 三種 signals 的責任邊界

```text
Kubernetes components + workloads
       | metrics      -> scrape -> time-series store -> alert/dashboard
       | logs         -> node agent -> log backend -> search/retention
       | OTLP traces  -> collector -> trace backend -> critical path
       + events/audit -> correlate with rollout, policy and identity
```

| Signal | 最擅長回答 | 主要限制 |
| --- | --- | --- |
| Metrics | 是否異常、何時開始、影響多大 | Aggregated；高 cardinality 會增加成本 |
| Logs | 發生了什麼、錯誤上下文為何 | 格式可能改變；volume 大且搜尋昂貴 |
| Traces | 一次 request 花在哪裡、跨 component 關係 | Sampling 會遺漏；需 context propagation |
| Kubernetes Events | Object lifecycle 的近期原因 | 有 TTL、會聚合，不是 audit trail |
| Audit logs | 誰對 API 做了什麼 | 成本與敏感資料風險，需獨立 policy |

實務調查通常從 symptom metric 找時間窗與 affected scope，再以 labels 對應 Pod/namespace/node/component，查 logs 的錯誤上下文，最後用 traces 拆解 latency path。Events、audit 與 deployment history 用來回答「當時改了什麼」。

## Metrics pipeline

Kubernetes system components 通常以 Prometheus text format 在 `/metrics` 暴露 metrics。kubelet 另有 `/metrics/cadvisor`、`/metrics/resource` 與 `/metrics/probes`。Scraping endpoint 不等於可觀測性完成；仍需 authentication/TLS、service discovery、retention、recording rules、alert routing 與 capacity planning。

常見 layers：

- **Resource metrics**：CPU/memory 的近期使用量，供 Metrics API、`kubectl top` 與 HPA 使用；不是長期歷史監控。
- **Component metrics**：API server、scheduler、controller manager、kubelet、etcd 等內部 behavior 與 latency。
- **Object-state metrics**：kube-state-metrics watch Kubernetes API，將 spec/status/conditions 轉成 metrics；它不量測 container CPU，也不屬於 Kubernetes core binary。
- **Application metrics**：request rate、error rate、latency、queue depth 與 business signals；Kubernetes platform metrics 不能取代它們。

```promql
# Symptom: Pods reported not ready
sum by (namespace) (kube_pod_status_ready{condition="false"})

# API request latency should use a histogram quantile over a rate window
histogram_quantile(
  0.99,
  sum by (le, verb) (rate(apiserver_request_duration_seconds_bucket[5m]))
)
```

Metric lifecycle 為 Alpha → Beta → Stable → Deprecated → Hidden → Deleted。Dashboard 與 alerts 應優先使用 stable metrics，upgrade 前掃描 deprecated metrics；不要用 `--show-hidden-metrics-for-version` 當長期解法。即使 metric name stable，labels 仍要看 stability contract，並控制 user-controlled labels 帶來的 cardinality。

## kube-state-metrics 的正確定位

kube-state-metrics 產生的是 API object state 的 snapshot series，例如 desired/available replicas、Pod phase、deletion timestamp、PVC condition。它適合回答「Kubernetes 認為 object 是什麼狀態」，不回答 process 為何慢或 container 實際用了多少 resources。

- Alert 應結合 duration，避免 transient reconciliation 造成 noise，例如 Deployment unavailable 持續數分鐘才觸發。
- Object labels/annotations allowlist 應最小化，避免 tenant-generated values 爆增 series。
- 它依賴 API LIST/WATCH；監控自身 scrape errors、watch errors、API latency 與 memory usage。
- Object state 可能 stale；incident 中要與直接 API read、Events 及 controller logs 交叉驗證。

## Logging architecture

Containerized application 最通用的做法是寫 structured records 到 `stdout`/`stderr`，由 runtime 寫入 node log files，再由 node-level agent（通常 DaemonSet）forward 到 cluster-external backend。Log storage 與 lifecycle 必須獨立於 Pod、container 與 node，否則 crash、eviction 或 node loss 後無法調查。

```text
app stdout/stderr
  -> container runtime log file + rotation
  -> node logging agent
  -> buffer / retry / backpressure
  -> centralized backend
  -> retention, access control, search and alerting
```

- Node agent 是一般預設；sidecar 只在需要轉換 legacy file format、分離 routing 或應用無法寫 stdout 時使用，因它增加 CPU/memory、storage 與 failure modes。
- Application 直接 push backend 會把 credentials、retry、buffering 與 vendor coupling 放進每個 workload，通常不是平台預設。
- Container log rotation 由 kubelet/runtime 與 node disk capacity共同決定。Collector outage 時必須有 bounded buffering，並監控 dropped records 與 ephemeral-storage pressure。
- Structured logs 應包含 timestamp、severity、service、environment、trace/span ID 與 stable workload identity；避免 secrets、tokens、完整 request bodies 與無界 user IDs。
- Multiline stack traces、rotation、backpressure、duplicate delivery 與 node reboot 都要測試；log transport 通常是 at-least-once，查詢與告警需容忍 duplicate。

## Kubernetes system logs

System component logs 可顯示 API requests、controller actions、scheduler decisions 與 kubelet events，但 log entries 與格式不受 Kubernetes API stability guarantees 保護。不要以固定字串 parsing 作為唯一 production alert；優先用 stable metrics、structured fields 與 version-aware parser。

- `-v` 增加 verbosity，也增加 CPU、I/O、storage cost 與洩漏敏感資料風險；incident 暫時提高後應還原。
- 非 GA 的 JSON system logging 不屬於本 production baseline，另見 [Kubernetes Observability Preview Features](observability-preview-features.md)。
- Linux 上 kubelet/container runtime 常由 journald 管理；static Pod control-plane components 則由 container logging path 收集。確認實際 distribution 與 managed-service access boundary。
- System logs、application logs 與 audit logs目的不同，應有不同 access、retention 與 redaction policy。

## Preview observability features

Kubernetes system-component tracing 與 JSON system logging 尚未全部 GA，不列入本 production baseline。其 OTLP data path、sampling、安全邊界與 adoption gate 見 [Kubernetes Observability Preview Features](observability-preview-features.md)。Application tracing 的 OpenTelemetry 設計不因此受限，但仍要獨立驗證 SDK、Collector 與 backend maturity。

## Production observability design

一套可防守的設計至少包含：

1. 由 SLO 與 failure modes 反推 signals，而不是「能收什麼就收什麼」。
2. Control plane、nodes、cluster add-ons、workloads 與 telemetry pipeline 本身都有 golden signals。
3. 統一 `cluster`、`namespace`、`workload`、`pod`、`node`、`service` 與 environment identity，同時控制 cardinality。
4. Metrics retention、log retention、trace sampling 依 incident window、compliance 與成本分層。
5. Alerts 有 owner、severity、runbook、silence policy 與 tested routing；dashboard 不是 alert strategy。
6. Telemetry backend/collector outage 不阻塞 application request path，並可觀測 dropped samples、queue saturation 與 ingestion lag。

## 故障排查範例：API latency 上升

```text
1. Metric: p99 API latency, request rate, inflight, APF queue/reject
2. Scope: verb, resource, priority level, client/user agent
3. Correlate: deploy/upgrade/config change and audit events
4. Trace: API server -> admission webhook / etcd / re-entrant request
5. Logs: timeout, throttling, leader change, etcd or webhook errors
6. Validate recovery: latency, error budget, queue depth and backlog drain
```

不要只看平均 latency。先區分 request execution 與 queue wait，再按 verb/resource/client 分解；如果所有 API calls 都慢，檢查 etcd、CPU 與 APF。如果特定 writes 慢，優先查 matching admission webhooks。如果只有 controller backlog，查 client throttling、retries 與 leader health。

## 常見面試題

### Metrics、logs 和 traces 能互相取代嗎？

不能。Metrics 適合偵測與量化，logs 提供離散事件和錯誤上下文，traces 顯示單次操作跨 components 的 critical path。高效率流程是以 metric 定位時間和 scope，再用 logs/traces 解釋原因。

### kube-state-metrics 和 Metrics Server 有何差異？

kube-state-metrics 將 Kubernetes API objects 的 desired/current state 暴露成 metrics；Metrics Server 聚合 node/kubelet 的近期 CPU/memory resource metrics，主要供 Metrics API、HPA 與 `kubectl top`。兩者資料來源、用途與 retention 都不同。

### 為何不對所有 traces 做 100% sampling？

高流量環境會造成 network、collector、storage 與 query 成本，telemetry 本身也可能影響 control plane。應依 SLO、錯誤/latency tail 與成本使用 head/tail sampling，並保留稀有 error paths 的策略。

### Kubernetes Events 是 logs 嗎？

Events 是 API objects 對近期 object lifecycle 的 best-effort 訊號，可能聚合且有 retention 限制；它們不是 durable application logs，也不是回答 actor/action 的 audit logs。

## 版本與內容新鮮度

- 本指南於 2026-08-21 依 Kubernetes v1.36 文件檢視。
- Alpha/Beta system telemetry 已移至 [Kubernetes Observability Preview Features](observability-preview-features.md)；本指南不將其當成 cluster-wide stable contract。
- kube-state-metrics 是 Kubernetes 生態系 add-on，不是 core component；需獨立安裝、升級與防護。
- 未保留依賴固定 log text、無界 cardinality、node-local-only retention 或 hidden deprecated metrics 的 legacy 做法。

## 參考資料

- [Observability](https://kubernetes.io/docs/concepts/cluster-administration/observability/)
- [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Metrics For Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/)
- [Metrics for Kubernetes Object States](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/)
- [System Logs](https://kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Traces For Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-traces/)

---

# Kubernetes Observability: Metrics, Logs, and Traces Interview Guide

Observability is not the installation of a dashboard. It is the ability to infer internal system state from metrics, logs, and traces, correlated with Kubernetes identity, changes, and service objectives. Each signal answers different questions. Collection without retention, cardinality control, alert ownership, and an incident workflow is not an operable observability system.

## Signal responsibility boundaries

```text
Kubernetes components + workloads
       | metrics      -> scrape -> time-series store -> alert/dashboard
       | logs         -> node agent -> log backend -> search/retention
       | OTLP traces  -> collector -> trace backend -> critical path
       + events/audit -> correlate with rollout, policy and identity
```

| Signal | Best question | Main limitation |
| --- | --- | --- |
| Metrics | Is it abnormal, when did it start, and how large is the impact? | Aggregated; high cardinality is costly |
| Logs | What happened and what was the error context? | Formats can change; volume and search are expensive |
| Traces | Where did one request spend time across components? | Sampling loses data; propagation is required |
| Kubernetes Events | What recently happened to an object? | TTL and aggregation; not an audit trail |
| Audit logs | Who did what to the API? | Cost and sensitive-data risk; needs a separate policy |

A practical investigation starts with a symptom metric to select the time window and affected scope, maps labels to Pod, namespace, node, and component identity, then inspects log context and traces the latency path. Events, audit, and deployment history answer what changed.

## Metrics pipeline

Kubernetes system components generally expose Prometheus text format at `/metrics`. Kubelet also exposes `/metrics/cadvisor`, `/metrics/resource`, and `/metrics/probes`. Scraping is not a complete observability solution; authentication/TLS, discovery, retention, recording rules, routing, and capacity planning remain necessary.

Common layers:

- **Resource metrics**: recent CPU/memory usage for the Metrics API, `kubectl top`, and HPA; not long-term historical monitoring.
- **Component metrics**: behavior and latency inside API server, scheduler, controller manager, kubelet, etcd, and other components.
- **Object-state metrics**: kube-state-metrics watches the API and converts spec, status, and conditions to metrics. It does not measure container CPU and is not a Kubernetes core binary.
- **Application metrics**: request rate, errors, latency, queue depth, and business signals. Platform metrics do not replace them.

```promql
# Symptom: Pods reported not ready
sum by (namespace) (kube_pod_status_ready{condition="false"})

# API request latency should use a histogram quantile over a rate window
histogram_quantile(
  0.99,
  sum by (le, verb) (rate(apiserver_request_duration_seconds_bucket[5m]))
)
```

The metric lifecycle is Alpha → Beta → Stable → Deprecated → Hidden → Deleted. Prefer stable metrics, scan dashboards and alerts for deprecations before upgrades, and do not treat `--show-hidden-metrics-for-version` as a permanent fix. Review label stability and control cardinality from user-controlled labels.

## Correctly positioning kube-state-metrics

kube-state-metrics produces snapshot series from API object state, such as desired/available replicas, Pod phase, deletion timestamps, and PVC conditions. It answers what Kubernetes believes about an object, not why a process is slow or how much resource a container consumes.

- Include duration in alerts to avoid noise from transient reconciliation.
- Minimize label and annotation allowlists to prevent tenant values from exploding series count.
- It depends on API LIST/WATCH. Monitor its scrape/watch errors, API latency, and memory.
- Object state can be stale; cross-check direct API reads, Events, and controller logs during incidents.

## Logging architecture

The common container pattern writes structured records to `stdout`/`stderr`; the runtime writes node log files, and a node agent, usually a DaemonSet, forwards them to an external backend. Log storage and lifecycle must be independent of Pods, containers, and nodes so evidence survives crashes, evictions, and node loss.

```text
app stdout/stderr
  -> container runtime log file + rotation
  -> node logging agent
  -> buffer / retry / backpressure
  -> centralized backend
  -> retention, access control, search and alerting
```

- A node agent is the normal default. Use a sidecar only to transform legacy file formats, separate routing, or support an application that cannot write stdout; it adds resource use and failure modes.
- Direct application push distributes credentials, retries, buffering, and vendor coupling into every workload and is rarely the platform default.
- Kubelet/runtime settings and node disk capacity jointly control rotation. Use bounded buffering and monitor drops and ephemeral-storage pressure during collector outages.
- Include timestamp, severity, service, environment, trace/span IDs, and stable workload identity. Exclude secrets, tokens, full bodies, and unbounded user identifiers.
- Test multiline parsing, rotation, backpressure, duplicate delivery, and reboot. Log transport is commonly at-least-once.

## Kubernetes system logs

System logs reveal API requests, controller actions, scheduler decisions, and kubelet events, but entries and formatting are outside Kubernetes API stability guarantees. Do not make fixed-text parsing the only production alert; prefer stable metrics, structured fields, and version-aware parsers.

- Higher `-v` verbosity costs CPU, I/O, and storage and can expose sensitive data. Revert temporary incident settings.
- Non-GA JSON system logging is outside this production baseline; see [Kubernetes Observability Preview Features](observability-preview-features.md).
- Linux kubelet and runtime logs often use journald, while static-Pod control-plane components follow container logging paths. Verify the distribution and managed-service boundary.
- System, application, and audit logs have different purposes and need separate access, retention, and redaction policies.

## Preview observability features

Kubernetes system-component tracing and JSON system logging are not fully GA and are outside this production baseline. Their OTLP path, sampling, security boundaries, and adoption gates are covered in [Kubernetes Observability Preview Features](observability-preview-features.md). Application tracing through OpenTelemetry remains a separate design whose SDK, Collector, and backend maturity must be validated independently.

## Production observability design

A defensible design includes:

1. Derive signals from SLOs and failure modes instead of collecting everything available.
2. Cover the control plane, nodes, add-ons, workloads, and the telemetry pipeline itself.
3. Normalize cluster, namespace, workload, Pod, node, service, and environment identity while controlling cardinality.
4. Tier metric retention, log retention, and trace sampling by incident window, compliance, and cost.
5. Give alerts an owner, severity, runbook, silence policy, and tested route. A dashboard is not an alert strategy.
6. Ensure collector/backend failure does not block application requests, and observe drops, saturation, and ingestion lag.

## Troubleshooting example: API latency increase

```text
1. Metric: p99 API latency, request rate, inflight, APF queue/reject
2. Scope: verb, resource, priority level, client/user agent
3. Correlate: deploy/upgrade/config change and audit events
4. Trace: API server -> admission webhook / etcd / re-entrant request
5. Logs: timeout, throttling, leader change, etcd or webhook errors
6. Validate recovery: latency, error budget, queue depth and backlog drain
```

Do not rely on average latency. Separate queue wait from execution, then split by verb, resource, and client. If all calls are slow, examine etcd, CPU, and APF. If only writes are slow, inspect matching webhooks. If only a controller is backlogged, inspect client throttling, retries, and leader health.

## Common interview questions

### Can metrics, logs, and traces replace one another?

No. Metrics detect and quantify, logs provide discrete events and error context, and traces show the cross-component critical path of one operation. An efficient workflow uses metrics to find time and scope, then logs and traces to explain cause.

### How do kube-state-metrics and Metrics Server differ?

kube-state-metrics exposes desired and current state from Kubernetes API objects. Metrics Server aggregates recent CPU/memory resource metrics from nodes and kubelets for the Metrics API, HPA, and `kubectl top`. Their data sources, use cases, and retention differ.

### Why not sample every trace?

At high volume, networking, collector, storage, and query cost can make telemetry harm the control plane. Use head or tail sampling based on SLOs, error and latency tails, cost, and a strategy for retaining rare failures.

### Are Kubernetes Events logs?

Events are best-effort API objects about recent object lifecycle, with aggregation and retention limits. They are neither durable application logs nor audit logs that prove actor and action.

## Version and freshness notes

- Reviewed on 2026-08-21 against Kubernetes v1.36 documentation.
- Alpha and beta system telemetry has moved to [Kubernetes Observability Preview Features](observability-preview-features.md); this guide does not present it as a cluster-wide stable contract.
- kube-state-metrics is an ecosystem add-on, not a core component, and requires separate installation, upgrades, and hardening.
- Legacy practices based on fixed log text, unbounded cardinality, node-local-only retention, or hidden deprecated metrics are omitted.

## References

- [Observability](https://kubernetes.io/docs/concepts/cluster-administration/observability/)
- [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Metrics For Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/)
- [Metrics for Kubernetes Object States](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/)
- [System Logs](https://kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Traces For Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-traces/)
