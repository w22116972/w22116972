# Kubernetes Observability Preview Features 延伸指南

本文件隔離 [Kubernetes Observability](observability.md) production baseline 以外、在 v1.36 仍未 GA 的 telemetry capabilities。

## Structured JSON system logging

Kubernetes system component 的 JSON log format 仍為 Alpha。它可改善 machine parsing，但 field、component coverage 與 rollout behavior 仍可能變動。實驗時必須使用支援 dual parsing 的 collector，保留 text fallback，並驗證 multiline、rotation、timestamp、severity 與 audit-log separation。

## System component tracing

Kubernetes system-component tracing 整體在 v1.36 仍為 Beta；kubelet tracing 已在 v1.34 stable，不代表每個 component 都已 GA。Tracing configuration 會把 spans 經 OTLP endpoint 送到 collector；應限制 sampling、保護 attributes 中的敏感資料，並避免 collector failure 反向影響 control plane。

採用前驗證 component-specific support、configuration API、TLS、backpressure、cardinality、retention 與 upgrade behavior。Trace 用來關聯 request path，不取代 metrics、logs、audit 或 SLO。

## 參考資料

- [System Logs](https://kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Traces for Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-traces/)

---

# Kubernetes Observability Preview Features Supplement

This guide isolates telemetry capabilities outside the [Kubernetes Observability](observability.md) production baseline that are not GA in v1.36.

## Structured JSON system logging

JSON log format for Kubernetes system components remains Alpha. It can improve machine parsing, but fields, component coverage, and rollout behavior may still change. Experiments need a collector that supports dual parsing, a text fallback, and validation for multiline handling, rotation, timestamps, severity, and separation from audit logs.

## System component tracing

Kubernetes system-component tracing overall remains Beta in v1.36. Kubelet tracing reached stable in v1.34, which does not mean that every component is GA. Tracing configuration sends spans to a collector through an OTLP endpoint; constrain sampling, protect sensitive attributes, and prevent collector failure from feeding back into the control plane.

Before adoption, verify component-specific support, configuration APIs, TLS, backpressure, cardinality, retention, and upgrade behavior. Traces correlate request paths; they do not replace metrics, logs, audit, or SLOs.

## References

- [System Logs](https://kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Traces for Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-traces/)
