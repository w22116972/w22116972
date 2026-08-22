# Envoy Gateway：面試深度指南

Envoy Gateway 是 Kubernetes-native API Gateway 與 Envoy Proxy control plane。它讓 platform team 使用 Kubernetes Gateway API 與 Envoy Gateway extension CRDs 宣告 routing、security、resilience 與 observability，而不必直接維護低階 Envoy xDS configuration。

## 一句話說清楚架構

```text
platform owner                 application owner
GatewayClass + EnvoyProxy      HTTPRoute / GRPCRoute / TLSRoute
          \                    /       + optional policies
           v                  v
       Envoy Gateway controller (control plane)
       watch -> validate -> translate -> status -> xDS
                              |
                              v
             Envoy Proxy fleet (data plane)
                              |
                   Service / Backend / endpoint
```

- Gateway API 是標準、role-oriented contract；Envoy Gateway 是實作該 contract 的 controller。
- Envoy Gateway controller 觀察 Kubernetes resources、建立 Envoy infrastructure，並將設定翻譯成 xDS。
- Envoy Proxy 才是接收 client connection、終止 TLS、套用 filters、選擇 upstream 與轉送 request 的 data plane。
- `Gateway`、`HTTPRoute` 已成功寫入 API server，不代表 data plane 已可用；必須檢查 status conditions、generated resources、xDS 與實際 traffic。

## 資源模型與 ownership boundary

| Resource | Owner | 主要責任 | 面試重點 |
| --- | --- | --- | --- |
| `GatewayClass` | infrastructure provider | 指定 `gateway.envoyproxy.io/gatewayclass-controller`，可透過 `parametersRef` 連結 `EnvoyProxy` | cluster-scoped；代表一種 gateway implementation |
| `Gateway` | platform/operator | 宣告 address、listeners、ports、protocols、TLS 與允許哪些 Routes attach | listener 是 traffic entry point，不是 application route |
| `HTTPRoute` / `GRPCRoute` | application team | host、path、header、method、backend、weight、filter | 透過 `parentRefs` attach，controller 以 status 回報是否接受 |
| `TLSRoute` / `TCPRoute` / `UDPRoute` | platform/application | L4 或 TLS passthrough routing | 必須確認所用 Gateway API channel 與 controller support |
| `ReferenceGrant` | 被引用 namespace 的 owner | 明確允許 cross-namespace reference | consumer 不能單方面取得跨 namespace 存取權 |
| `BackendTLSPolicy` | service owner/platform | 驗證 gateway-to-backend TLS certificate 與 hostname | backend encryption 必須驗證身份，不只是啟用 TLS |
| `EnvoyProxy` | platform/operator | 調整 Envoy deployment/service、provider、telemetry、bootstrap 與 infrastructure | 它管理 data-plane shape，不負責 application routing |
| `ClientTrafficPolicy` | platform/operator | downstream client 到 listener 的 connection/TLS/HTTP behavior | target `Gateway` 或 listener，屬於 downstream side |
| `BackendTrafficPolicy` | platform/application | timeout、retry、circuit breaker、rate limit、load balancing、health check、failover | target `Gateway`/Route，屬於 upstream behavior |
| `SecurityPolicy` | security/platform/application | authentication、authorization、CORS、CSRF、credential injection | scope 與 precedence 會直接改變 security outcome |
| `Backend` | platform/application | 表達 FQDN、IP 或 Unix Domain Socket 等非一般 Service backend | 仍需治理 DNS、TLS、egress 與 availability |
| `EnvoyExtensionPolicy` / `EnvoyPatchPolicy` | platform owner | 加入 extension 或修改 xDS | 高風險 escape hatch，應限制權限與測試 rollback |

`GatewayClass -> Gateway -> Route -> backend` 是主要 attachment chain。Route 與 Gateway 不同 namespace 時，先由 Gateway listener 的 `allowedRoutes` 決定 Route 是否可 attach；Route 若引用另一 namespace 的 Service、Secret 或其他 object，通常還需要 target namespace 的 `ReferenceGrant`。

## 最小 HTTP routing 範例

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: gateway-system
spec:
  gatewayClassName: envoy
  listeners:
    - name: https
      hostname: api.example.com
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: api-example-com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: public
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
  namespace: application
spec:
  parentRefs:
    - name: public
      namespace: gateway-system
      sectionName: https
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api
          port: 8080
```

Production review 不只看 YAML syntax：確認 hostname intersection、listener protocol、`allowedRoutes`、`ReferenceGrant`、backend port、Route status 與 certificate scope。

## Policy attachment、scope 與 precedence

Envoy Gateway policies 可使用 `targetRefs` 精確指定 resource，或以 `targetSelectors` 依 labels 選取。`sectionName` 可縮小到特定 listener 或 route rule。選擇器方便大規模套用，但 label 變更可能同時改變許多 resources 的 effective policy，因此需要 admission control、ownership 與 change review。

面試時不要只回答「最接近 Route 的 policy wins」：

- `SecurityPolicy` 通常依 route rule、route、listener、Gateway 的 specificity 決定；未設定 `mergeType` 時不會自動合併，只採最 specific configuration。
- `SecurityPolicy` 可在 child policy 使用 `StrategicMerge` 與 parent policy 合併；相同 feature 仍由 child 覆蓋，Secret/backend reference 依最初宣告該欄位的 policy namespace 解析。
- `ClientTrafficPolicy` 以 listener specificity 為主；同一 specificity 有衝突時，較早建立的 policy 優先，再以 namespaced name 排序。
- `BackendTrafficPolicy` 套用於 Gateway、`HTTPRoute` 或 `GRPCRoute`；Route/rule-specific intent 應避免與 broad selector policy 意外重疊。
- 一定要讀 policy 與 target resource 的 status；「object exists」不等於「attached and enforced」。

## TLS 與 trust boundary

需要分開四種情境：

1. Client 到 Gateway TLS termination：`Gateway` listener 引用 certificate Secret；Envoy 解密後執行 L7 routing/policies。
2. TLS passthrough：`TLSRoute` 依 SNI 轉送 encrypted stream；Gateway 看不到 HTTP path/header，因此無法執行相同的 L7 features。
3. Gateway 到 backend TLS：用 `BackendTLSPolicy` 驗證 hostname 與 CA trust；不要把 `insecureSkipVerify` 當 production solution。
4. mTLS：downstream mTLS 驗證 external client；backend mTLS 則讓 Envoy 以 client certificate 向 upstream 驗證。兩段是不同 trust relationship。

TLS certificate 可由 Secret、cert-manager 或支援的 private key/SDS provider 管理。Production 應限制 Secret RBAC、定義 renewal/rotation、監控 expiry，並驗證 rotation 不造成 listener outage。Envoy admin、pprof 與 config dump 可能暴露敏感 configuration，預設僅綁 localhost，透過受控 `port-forward` 存取。

## Authentication 與 authorization

`SecurityPolicy` 支援 API key、Basic Auth、JWT、OIDC、external authorization、client mTLS、IP/CIDR、header/method、JWT claims、GeoIP、CORS、CSRF 與 credential injection 等能力。設計時要區分：

- Authentication 證明 caller identity；authorization 決定該 identity 是否能執行 action。
- CORS 是 browser enforcement contract，不是 API authentication 或 server-side authorization。
- CSRF 防護主要針對 browser 自動攜帶 credential 的 state-changing requests。
- `X-Forwarded-For` 只有在 trusted proxy hops 正確設定時才可作 client IP 決策；否則 attacker 可偽造 header。
- JWT signature/issuer/audience validation 不等於 business authorization；仍需檢查 claims、tenant 與 action。
- OIDC redirect/session flow 與 machine-to-machine JWT/API key flow 的 failure mode 不同。
- External auth 或 OPA sidecar 是 request path dependency；必須定義 timeout、failure mode、capacity 與 observability。
- Credential injection 可集中管理 upstream credential，但 gateway 因此成為高價值 trust boundary，應限制 policy/Secret write access。

## Traffic management 與 resilience

### Routing 與 transformation

Envoy Gateway 支援 HTTP host/path/header/method routing、redirect、URL rewrite、request/response header modification、traffic splitting、request mirroring、direct response、response override、compression、buffering、session persistence，以及 HTTP/3、gRPC、TCP、UDP、TLS 與 HTTP CONNECT 等 use cases。

- Weighted `backendRefs` 可做 canary，但 rollout 必須同時看 error rate、latency、saturation 與 rollback condition。
- Mirroring 不應影響 primary response，但 mirrored backend 仍會收到可能敏感或具有 side effect 的 request。
- Rewrite 改變送往 upstream 的 URI；redirect 要 client 發出新 request，語意與 latency 不同。
- Request buffering 會增加 memory/disk pressure，對 streaming 或大型 upload 不一定合適。
- Session persistence 降低 load distribution flexibility；優先讓 application stateless。

### Timeout、retry、circuit breaker 與 failover

- Timeout 限制 client、route 或 upstream operation 最長等待時間，是 latency budget 的邊界。
- Retry 只適合可安全重試的 failure 和 idempotent operation；要設定 attempts、per-try timeout 與 backoff，避免 retry storm。
- Circuit breaker 限制 connection、pending request、active request 或 retry，保護 backend，但門檻太低會把容量問題放大成 rejection。
- Active/passive health check 決定 endpoint 是否 eligible；Kubernetes readiness 與 Envoy upstream health 是不同 signal。
- Failover/priority routing 應搭配 outlier detection、capacity reservation 與 recovery test；宣告 secondary backend 不代表 failover 一定可承載流量。
- Fault injection 只應用於受控測試 scope，不能意外套到 production-wide Gateway。

### Load balancing 與 locality

Envoy Gateway 的 `BackendTrafficPolicy` 可設定 `RoundRobin`、`Random`、`LeastRequest`、`ConsistentHash` 與使用 ORCA metrics 的 `BackendUtilization`；reference snapshot 指出預設為 `LeastRequest`。Consistent hash 提供 affinity，不保證狀態永遠落在同一 endpoint。Zone-aware routing 可 prefer local 或設定 weighted zones，但它是 routing policy，不取代 multi-zone capacity、topology spread 與 failover validation。

### Local 與 global rate limiting

| 類型 | Enforcement scope | Dependency | 適用情境 |
| --- | --- | --- | --- |
| Local | 每個 Envoy replica 各自計數 | Envoy local state | 低 latency、先擋 burst/abuse；總量會隨 replicas 改變 |
| Global | 整個 Envoy fleet 協調計數 | Rate Limit Service 與通常為 Redis 的 datastore | tenant quota、shared backend protection、全域一致 limit |

兩者可以同時啟用：local 先執行，global 再判斷，request 必須通過兩層。Rate limit key 若直接使用高 cardinality 或可偽造 header，會帶來 correctness、memory 與 abuse 問題。

## Deployment 與 lifecycle

推薦使用 pinned version 的 Helm chart 或受 GitOps 管理的 manifests；不要在 production 長期依賴 `latest` URL/image。Reference set 同時包含 Helm、YAML、`egctl`、Argo CD、Flux、air-gapped installation、自訂 control-plane certificate，以及拆分 CRDs/core/addons charts 的方法。

Production checklist：

- 將 Gateway API CRDs、Envoy Gateway CRDs、controller 與 addons 視為有順序的 lifecycle；升級前檢查 API compatibility、conversion 與 rollback。
- GitOps 安裝要處理 CRD ordering、health assessment 與 controller readiness；不要只看到 sync 成功。
- Air-gapped environment 必須 mirror controller、Envoy、rate-limit/extension images 及 charts，並重寫所有 image references。
- `EnvoyProxy` 可控制 replicas、resources、pod/service annotations、load balancer、telemetry 與 shutdown behavior；避免 application team 任意修改 platform-wide data plane。
- Gateway namespace mode、standalone mode 與其他 deployment modes 改變 watch scope、resource placement、tenancy與 blast radius；選擇前先定義 ownership 與 isolation requirement。
- Graceful shutdown 要讓 load balancer 停止送新流量、Envoy drain existing connections，且 Pod `terminationGracePeriodSeconds` 足以完成 drain。
- Control plane certificate replacement 必須驗證雙向 trust 與 rotation ordering，避免 controller 與 managed Envoy 失去連線。

Ingress migration 可用 `ingress2gateway` 或 reference 中的 `ingress2eg` 輔助轉換，但工具輸出只是起點。逐項比較 annotations、path matching、regex/rewrite、authentication、CORS、rate limits、source IP、TLS、health check、load balancer behavior 與 observability；以 parallel endpoint/canary 驗證再切 DNS，保留 rollback。

## Extensibility：從穩定 API 到 escape hatch

依風險由低到高選擇：

1. Gateway API standard fields/filters。
2. Envoy Gateway typed policies 與 `HTTPRouteFilter`。
3. External auth、external processing、Lua、Wasm、dynamic module 或 OPA sidecar/UDS。
4. Extension server 修改 translation pipeline。
5. `EnvoyPatchPolicy` 直接 patch xDS。

越往下 portability、upgrade safety 與 supportability 越低。Wasm image 要做 supply-chain scanning/signing；Lua/Wasm/ext-proc 要限制 CPU、memory、timeout 與 failure mode；dynamic module 與 Envoy binary ABI/version 相容性必須測試；xDS patch 應有 golden tests、canary 與快速 rollback。Remote infrastructure provider 與 standalone deployment 會改變 controller 和 data-plane lifecycle boundary，也需獨立 threat model。

## Observability：分開看 control plane 與 data plane

| 訊號 | 回答的問題 | 常見來源 |
| --- | --- | --- |
| Gateway API status | desired state 是否被接受與解析 | `GatewayClass`, `Gateway`, Route, Policy conditions |
| Control-plane logs/metrics | watch、translation、xDS、infrastructure reconciliation 是否正常 | Envoy Gateway logs、admin console、exported metrics |
| Data-plane access logs | request match 到哪條 route、回應碼與 upstream 是什麼 | Envoy access log |
| Proxy metrics | traffic、latency、retry、reset、upstream health、rate limit | Prometheus / Grafana |
| Distributed traces | request 跨 gateway/backend 的 latency 與 failure 在哪裡 | OpenTelemetry-compatible tracing |
| Health-check logs | endpoint 為何被判定 healthy/unhealthy | Envoy health-check log |

Gateway API metadata 應加入 metrics/logs，方便由 hostname、Gateway、Route 與 backend 追查 ownership。不要只監控 aggregate 2xx；至少觀察 4xx/5xx、p95/p99 latency、active/pending connections、upstream resets、retry amplification、rate-limit decisions、certificate expiry 與 config rejection。

## Troubleshooting 順序

```text
DNS / external LB address
  -> GatewayClass Accepted
  -> Gateway Accepted + Programmed
  -> Route Accepted + ResolvedRefs
  -> policy status / ReferenceGrant
  -> generated Envoy Deployment, Service, Pods
  -> Service / EndpointSlice / external Backend
  -> access log and proxy metrics
  -> xDS config_dump / stats only when needed
```

常用 read-only commands：

```bash
kubectl get gatewayclass,gateway,httproute,grpcroute -A
kubectl get gateway public -n gateway-system -o yaml
kubectl get httproute api -n application -o yaml
egctl x status all -A
kubectl get deploy,svc,pod -n envoy-gateway-system
kubectl logs -n envoy-gateway-system deploy/envoy-gateway
```

- `Accepted=True` 但 `ResolvedRefs=False` 常表示 backend/Secret 不存在、kind/group/port 錯誤或缺少 `ReferenceGrant`。
- `Gateway Programmed=False` 先查 GatewayClass/controller、address provisioning、listener conflict 與 generated infrastructure。
- 404 常是 listener/hostname/path 沒 match；503 多查 endpoints、health、TLS、connection；reference 中 invalid route 可能被轉成 `direct_response` 500，需用 access log 的 `response_code_details` 判斷。
- Application developer 通常應使用 exported telemetry；只有 platform admin 才需要受控地 port-forward control-plane admin console 或 Envoy admin interface 的 localhost `19000`，查看 stats/config dump。
- 不要從單一 HTTP status 猜 root cause；沿 desired state、translation、data plane、backend 四層建立證據。

## 面試題與答題骨架

### Envoy Gateway、Gateway API 與 Envoy Proxy 有何不同？

Gateway API 是 Kubernetes networking contract；Envoy Gateway 是 controller/control plane，負責 reconcile、infrastructure 與 xDS translation；Envoy Proxy 是處理 live traffic 的 data plane。

### Gateway API 為何比 Ingress 更適合 platform engineering？

它以 `GatewayClass`、`Gateway`、Route 與 policy attachment 分離 infrastructure provider、operator 與 application developer，提供更明確的 protocol、cross-namespace permission、status 與 portability。Ingress frozen 並不等於必須立即移除；migration 仍取決於 feature parity 與風險。

### 如何設計 production TLS？

先說清楚 downstream termination/passthrough 與 upstream TLS/mTLS；再說 Secret/cert-manager ownership、CA 與 hostname validation、rotation、minimum TLS、RBAC、expiry monitoring，以及禁止用 skip verification 當永久方案。

### Retry 如何造成 outage？

如果 request 不具 idempotency、per-try timeout 太長或每層都 retry，一次 backend failure 會被放大成 retry storm。使用有限 attempts、exponential backoff、overall deadline、circuit breaker、retry budget 與 metrics 驗證。

### 如何安全擴充 Envoy？

優先 standard API 與 typed CRDs；再依需求選 ext-auth/ext-proc/Wasm。直接 patch xDS 是最後手段，因為 validation、portability、upgrade compatibility 與 rollback 成本最高。

## Reference coverage 與 freshness

本指南已檢視 [`refs/envoy-gateway`](../../../refs/envoy-gateway/) 下的全部 Markdown references，並按以下方式整合：

- `concepts/`：API gateway、Gateway API、proxy、load balancing、rate limiting 與三種核心 policy models。
- `api/`：GatewayClass、Gateway、HTTPRoute、GRPCRoute、ReferenceGrant、BackendTLSPolicy 與 Envoy Gateway extension types；API reference 用來確認欄位語意，不逐欄複製。
- `install/` 與 `boilerplates/`：Helm/YAML/egctl、Argo CD、Flux、custom certificates、CRD/core/addons charts、migration、prerequisites、testing 與 rollout。
- `tasks/traffic/`：HTTP/gRPC/TCP/UDP/TLS/HTTP3 routing、header/rewrite/redirect、splitting/mirroring、timeouts/retries/circuit breaking、health/failover、rate/bandwidth/connection limits、load balancing/locality、buffering/compression/persistence 與 external/multicluster backends。
- `tasks/security/`：TLS/mTLS、cert-manager/private keys、API key/Basic/JWT/OIDC/ext-auth、claim/header/IP/GeoIP authorization、CORS/CSRF、credential injection 與 threat model。
- `tasks/observability/`：Gateway/Route metadata and metrics、proxy logs/metrics/traces、health-check/rate-limit telemetry 與 Grafana integration。
- `tasks/operations/`：deployment modes、Gateway namespace/standalone mode、EnvoyProxy customization、air-gap、graceful shutdown 與 `egctl`。
- `tasks/extensibility/`：Wasm、Lua、dynamic modules、ext-proc、OPA UDS、extension server、remote infrastructure provider 與 Envoy patch policy。
- `troubleshooting/`：resource conditions、traffic evidence、control-plane admin console 與 Envoy Proxy admin interface。

這些 local references 看起來是特定 Envoy Gateway documentation snapshot，含 `v1alpha1` extension APIs、experimental Gateway API types、template placeholders 與 version-dependent behavior。實作前應以部署版本的 CRDs、Gateway API conformance/support table、Helm chart values 與 release notes 為準；不要直接複製 reference 中的 `latest` URLs、image tags 或未替換的 `{{< ... >}}` placeholders。

---

# Envoy Gateway: Interview-Depth Guide

Envoy Gateway is a Kubernetes-native API gateway and an Envoy Proxy control plane. It lets platform teams declare routing, security, resilience, and observability through the Kubernetes Gateway API and Envoy Gateway extension CRDs instead of maintaining low-level Envoy xDS configuration directly.

## Explain the architecture in one answer

```text
platform owner                 application owner
GatewayClass + EnvoyProxy      HTTPRoute / GRPCRoute / TLSRoute
          \                    /       + optional policies
           v                  v
       Envoy Gateway controller (control plane)
       watch -> validate -> translate -> status -> xDS
                              |
                              v
             Envoy Proxy fleet (data plane)
                              |
                   Service / Backend / endpoint
```

- Gateway API is the standard, role-oriented contract; Envoy Gateway is a controller that implements it.
- The Envoy Gateway controller watches Kubernetes resources, creates Envoy infrastructure, and translates configuration into xDS.
- Envoy Proxy is the data plane that accepts connections, terminates TLS, runs filters, selects upstreams, and forwards requests.
- Successfully storing a `Gateway` or `HTTPRoute` does not prove that traffic works. Verify status conditions, generated infrastructure, xDS delivery, and real requests.

## Resource model and ownership boundaries

| Resource | Owner | Primary responsibility | Interview point |
| --- | --- | --- | --- |
| `GatewayClass` | infrastructure provider | Select the Envoy Gateway controller and optionally reference `EnvoyProxy` parameters | Cluster-scoped representation of an implementation |
| `Gateway` | platform/operator | Declare addresses, listeners, ports, protocols, TLS, and allowed Routes | A listener is an entry point, not an application route |
| Route resources | application team | Match traffic and select weighted backends and filters | Attach through `parentRefs`; status reports acceptance |
| `ReferenceGrant` | owner of referenced namespace | Authorize a cross-namespace reference | The consumer cannot grant itself access |
| `BackendTLSPolicy` | service owner/platform | Verify backend certificate identity and trust | Encryption without identity validation is insufficient |
| `EnvoyProxy` | platform/operator | Shape proxy deployment, Service, provider, telemetry, bootstrap, and infrastructure | Controls the data-plane form, not application routing |
| `ClientTrafficPolicy` | platform/operator | Configure downstream client-to-listener behavior | Applies on the downstream side |
| `BackendTrafficPolicy` | platform/application | Configure timeout, retry, circuit breaking, rate limiting, balancing, health, and failover | Applies on the upstream side |
| `SecurityPolicy` | security/platform/application | Configure authentication, authorization, CORS, CSRF, and credential injection | Scope and precedence determine the effective security posture |
| `Backend` | platform/application | Represent FQDN, IP, or Unix Domain Socket backends | External backends still require DNS, TLS, egress, and availability controls |
| Extension and patch policies | platform owner | Extend filters or patch xDS | High-risk escape hatches requiring strict RBAC and rollback tests |

The primary attachment chain is `GatewayClass -> Gateway -> Route -> backend`. For cross-namespace Routes, the Gateway listener's `allowedRoutes` governs attachment. A Route referencing a Service, Secret, or another object in a different namespace usually also requires a `ReferenceGrant` owned by the target namespace.

## Minimal HTTP routing example

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: gateway-system
spec:
  gatewayClassName: envoy
  listeners:
    - name: https
      hostname: api.example.com
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: api-example-com
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: public
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
  namespace: application
spec:
  parentRefs:
    - name: public
      namespace: gateway-system
      sectionName: https
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api
          port: 8080
```

A production review must go beyond YAML syntax. Verify hostname intersection, listener protocol, `allowedRoutes`, `ReferenceGrant`, backend port, Route status, and certificate scope.

## Policy attachment and precedence

Policies may use `targetRefs` for exact resources or `targetSelectors` for label-based selection. `sectionName` can narrow attachment to a listener or route rule. Selectors are operationally powerful, but a label change can alter effective policy across many resources, so admission controls, ownership, and review are necessary.

- `SecurityPolicy` generally resolves specificity from route rule to route to listener to Gateway. Without `mergeType`, only the most specific configuration applies.
- A child `SecurityPolicy` can use `StrategicMerge`; overlapping features are still won by the child, while Secret and backend references remain relative to the namespace of the policy that originally supplied the field.
- `ClientTrafficPolicy` favors listener specificity. At equal specificity, the older policy wins, followed by namespaced-name ordering.
- `BackendTrafficPolicy` targets a Gateway or HTTP/gRPC Route. Avoid accidental overlap between route-specific intent and broad selector policies.
- Always inspect policy and target status. Object existence is not proof of attachment or enforcement.

## TLS and security boundaries

Separate four designs: downstream TLS termination, TLS passthrough, gateway-to-backend TLS, and mTLS. Passthrough preserves encryption to the backend but prevents HTTP-level routing and policy at the gateway. Backend TLS should validate CA trust and hostname through `BackendTLSPolicy`; `insecureSkipVerify` is not a production solution. Downstream and backend mTLS establish different trust relationships.

`SecurityPolicy` covers API key, Basic Auth, JWT, OIDC, external authorization, client mTLS, IP/CIDR, header/method, claim and GeoIP authorization, CORS, CSRF, and credential injection. Authentication proves identity; authorization permits an action. CORS is a browser contract, not API authorization. JWT validation is not business authorization. Client-IP policy is valid only when trusted proxy hops and forwarded headers are correctly configured.

External authorization and OPA become request-path dependencies, so define timeout, failure behavior, capacity, and telemetry. Credential injection centralizes upstream credentials but turns the gateway into a high-value trust boundary. Restrict policy and Secret writers.

Certificates may come from Secrets, cert-manager, or supported private-key/SDS providers. Restrict Secret RBAC, define renewal and rotation procedures, monitor expiry, and prove that rollover does not interrupt listeners. Admin, pprof, and config-dump endpoints may expose sensitive configuration; keep them on localhost and access them only through controlled port forwarding.

## Authentication and authorization

- Authentication establishes caller identity; authorization decides whether that identity may perform an action.
- CORS is browser enforcement metadata, not authentication or server-side authorization.
- CSRF protection mainly addresses state-changing requests where browsers automatically attach credentials.
- `X-Forwarded-For` is safe for client-IP decisions only when the trusted proxy-hop model is correct.
- JWT signature, issuer, and audience validation do not replace claim-, tenant-, and action-level authorization.
- OIDC browser sessions and machine-to-machine JWT/API-key flows have different failure and revocation models.
- External auth must have explicit timeout, failure-open/failure-closed, capacity, and telemetry decisions.

## Traffic management and resilience

Envoy Gateway supports HTTP host/path/header/method routing, redirects, rewrites, header changes, weighted splitting, mirroring, direct responses, response overrides, compression, buffering, persistence, HTTP/3, gRPC, TCP, UDP, TLS, and HTTP CONNECT.

- A rewrite changes the upstream URI; a redirect asks the client to issue another request.
- Mirrored traffic can still expose sensitive data or trigger side effects.
- Buffering consumes resources and may be unsuitable for streaming or large uploads.
- Persistence reduces balancing flexibility; prefer stateless applications.
- Retries require idempotency, bounded attempts, per-try timeout, backoff, and a total deadline to prevent retry storms.
- Circuit breakers protect upstreams through connection/request limits but can amplify rejection when mis-sized.
- Kubernetes readiness and Envoy active/passive upstream health are distinct signals.
- Failover needs outlier detection, reserved capacity, and tested recovery; a secondary backend declaration does not prove capacity.

`BackendTrafficPolicy` supports `RoundRobin`, `Random`, `LeastRequest`, `ConsistentHash`, and ORCA-based `BackendUtilization`; the reference snapshot identifies `LeastRequest` as the default. Local rate limiting counts independently per Envoy replica, while global rate limiting coordinates the fleet through a Rate Limit Service and usually Redis. Both can be layered: local evaluation occurs first, and the request must pass both.

Zone-aware routing may prefer local endpoints or apply explicit zone weights, but it does not replace multi-zone capacity, topology spread, or failover validation. Rate-limit keys based on high-cardinality or untrusted headers can cause correctness, memory, and abuse problems.

## Deployment and lifecycle

Use a pinned Helm chart or GitOps-managed manifests. Treat Gateway API CRDs, Envoy Gateway CRDs, the controller, and addons as an ordered lifecycle. Validate API compatibility, CRD conversion, controller readiness, and rollback. Air-gapped deployments must mirror every controller, proxy, rate-limit, and extension artifact.

Deployment mode, Gateway namespace mode, and standalone mode change watch scope, resource placement, tenancy, and blast radius. `EnvoyProxy` controls replicas, resources, annotations, load-balancer configuration, telemetry, and shutdown behavior. Graceful shutdown requires upstream load balancers to stop new traffic and a termination grace period long enough for Envoy draining.

GitOps installation must handle CRD ordering and controller health rather than treating sync success as runtime readiness. Control-plane certificate replacement must preserve mutual trust throughout the rotation. Keep application teams from changing platform-wide proxy infrastructure without an explicit ownership model.

Ingress conversion tools can bootstrap migration, but their output is not proof of parity. Compare annotations, matching, rewrites, authentication, CORS, rate limits, source IP, TLS, health checks, load balancer behavior, and telemetry. Validate through a parallel endpoint or canary before DNS cutover and retain a rollback path.

## Extensibility: from stable API to escape hatch

Choose extension mechanisms in increasing order of risk: standard Gateway API, typed Envoy Gateway policies, external auth/processing or Lua/Wasm, dynamic modules or extension server, and finally raw xDS patching. Higher levels reduce portability and upgrade safety. Require artifact signing/scanning, resource limits, timeout and failure semantics, compatibility tests, canaries, and rollback.

Wasm images need supply-chain controls; Lua, Wasm, and external processing need CPU, memory, timeout, and failure-mode limits; dynamic modules require binary and ABI compatibility with Envoy. Extension servers alter translation behavior, and raw xDS patches need golden tests. Remote infrastructure providers and standalone control planes also change lifecycle and trust boundaries and deserve a separate threat model.

## Observability: separate control plane and data plane

Separate control-plane and data-plane evidence. Gateway API conditions explain acceptance and reference resolution; controller logs and metrics explain reconciliation and xDS; Envoy access logs, metrics, traces, and health-check logs explain live traffic.

Attach Gateway API metadata to metrics and logs so teams can trace a hostname through Gateway, Route, and backend ownership. Monitor more than aggregate success rate: include 4xx/5xx, p95/p99 latency, active and pending connections, upstream resets, retry amplification, rate-limit decisions, certificate expiry, and configuration rejection.

## Troubleshooting order

```text
DNS / external LB address
  -> GatewayClass Accepted
  -> Gateway Accepted + Programmed
  -> Route Accepted + ResolvedRefs
  -> policy status / ReferenceGrant
  -> generated Envoy Deployment, Service, Pods
  -> Service / EndpointSlice / external Backend
  -> access log and proxy metrics
  -> xDS config_dump / stats only when needed
```

`Accepted=True` with `ResolvedRefs=False` commonly indicates a missing backend or Secret, a wrong kind/group/port, or a missing `ReferenceGrant`. For `Programmed=False`, inspect the class/controller, address provisioning, listener conflicts, and generated infrastructure. A 404 often indicates no listener/host/path match, while 503 points toward endpoints, health, TLS, or connectivity. Use `response_code_details` rather than guessing from status alone.

```bash
kubectl get gatewayclass,gateway,httproute,grpcroute -A
kubectl get gateway public -n gateway-system -o yaml
kubectl get httproute api -n application -o yaml
egctl x status all -A
kubectl get deploy,svc,pod -n envoy-gateway-system
kubectl logs -n envoy-gateway-system deploy/envoy-gateway
```

Application developers should normally rely on exported telemetry. Platform administrators may use controlled port forwarding to the localhost-only control-plane console or Envoy admin interface to inspect metrics and `config_dump`. Never expose admin or pprof endpoints publicly.

## Interview answer framework

- Envoy Gateway versus Envoy Proxy: control plane and reconciler versus traffic-handling data plane.
- Gateway API versus Ingress: role separation, typed routes, explicit cross-namespace permission, portable status, and extensibility; frozen Ingress does not require an emergency migration.
- Production TLS: separate downstream and upstream trust, validate CA and hostname, automate rotation, restrict Secret RBAC, monitor expiry, and test rollover.
- Safe retries: idempotency, bounded attempts, backoff, per-try and total deadlines, circuit breakers, retry budgets, and amplification metrics.
- Safe extensibility: prefer standard or typed APIs; raw xDS patches are last because validation, portability, upgrades, and rollback become harder.

## Reference coverage and freshness

This guide reviewed every Markdown file under [`refs/envoy-gateway`](../../../refs/envoy-gateway/). It consolidates concepts and API references; installation and boilerplates; traffic, security, observability, operations, and extensibility tasks; and all troubleshooting material. The API reference was used to verify semantics rather than copied field by field.

The local references appear to be a version-specific documentation snapshot and include `v1alpha1` extension APIs, experimental Gateway API types, template placeholders, and version-dependent behavior. Before implementation, verify the installed CRDs, Gateway API conformance/support table, chart values, and release notes for the deployed version. Do not copy `latest` URLs, image tags, or unresolved `{{< ... >}}` placeholders into production manifests.
