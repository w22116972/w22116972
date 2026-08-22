# Kubernetes 網路：面試深度指南

Kubernetes networking 的核心不是某一個 proxy，而是一組互相銜接的 contracts：CNI 提供 Pod connectivity，Service 提供穩定的虛擬入口，EndpointSlice 發布實際 backends，DNS 提供名稱解析，Ingress 或 Gateway API 處理 north-south traffic，而 NetworkPolicy 限制 L3/L4 traffic。實際 data plane 可能由 kube-proxy、eBPF、cloud load balancer 或其他 controller 實作。

## 先掌握封包與控制流程

```text
east-west traffic
client Pod
   | DNS: api.prod.svc.cluster.local -> Service ClusterIP
   v
Service virtual IP / data plane
   | watches EndpointSlices and selects a ready endpoint
   v
backend Pod IP:targetPort

north-south traffic
external client
   v
cloud LB / edge proxy
   v
Ingress or Gateway controller data plane
   v
Service -> EndpointSlice -> ready backend Pod

policy path
NetworkPolicy API object -> supporting CNI / policy engine -> L3/L4 enforcement
```

控制面與資料面必須分開回答：

- `Service`、`EndpointSlice`、`Ingress`、Gateway API resources 與 `NetworkPolicy` 是 desired state/API contracts。
- controllers 觀察 API objects，建立 load balancer、route 或 endpoint metadata。
- kube-proxy、CNI/eBPF dataplane、Ingress/Gateway proxy 與 cloud load balancer 才實際轉送或阻擋 packets。
- API object 已建立不代表路徑已可用；要確認 controller、status、EndpointSlice、DNS 與實際連線。

## Kubernetes 網路模型與責任邊界

面試時可先說出 Kubernetes 對 cluster network 的基本期待：每個 Pod 有自己的 IP；Pods 應能直接以 Pod IP 溝通，而不要求 application 自己處理 NAT。Kubernetes 定義 API 與行為，但不內建完整 Pod network；cluster 必須安裝符合需求的 CNI/network implementation。

| 層次 | 穩定抽象 | 主要責任 | 不應混淆的事項 |
| --- | --- | --- | --- |
| Pod connectivity | Pod IP、CNI | 配置 interface、routes、IPAM 與跨 node connectivity | CNI 不等於一定支援 NetworkPolicy |
| Service discovery | Service、DNS | 提供穩定 VIP/name，解耦 client 與短生命週期 Pod IP | Service 不保證 application healthy |
| Backend discovery | EndpointSlice | 發布 ready/serving/terminating endpoints、zone 與 address family | 不應再依賴 legacy Endpoints API |
| Edge routing | Ingress / Gateway API | TLS、host/path/protocol routing 與外部入口 | Resource 本身不會轉送流量，必須有 controller |
| Segmentation | NetworkPolicy | 對選定 Pods 套用 ingress/egress L3/L4 allow rules | 不是 L7 authorization、WAF 或 universal firewall |

## Service：穩定入口，不是 Pod lifecycle manager

Service 以 selector 找出 Pods，control plane 再建立 EndpointSlices。Client 使用 Service DNS name 或 virtual IP，資料面將 connection 導向可用 endpoint。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app.kubernetes.io/name: api
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
```

- `port` 是 Service 對 client 暴露的 port；`targetPort` 是 backend endpoint 接收流量的 port；`nodePort` 只適用於需要 node-level port 的 Service types。
- `ClusterIP` 是預設類型，只在 cluster network 內提供 virtual IP。
- `NodePort` 在每個 node 開啟同一 port，並建立在 ClusterIP 之上；通常是外部 load balancer 的 building block，不是優先的 public edge design。
- `LoadBalancer` 要由 cloud/provider controller 或其他 implementation 建立外部 load balancer；Kubernetes 本身不提供該設備。
- `ExternalName` 只透過 DNS CNAME 將名稱映射到另一個 DNS name，沒有 selector、proxy 或 health check；HTTP/TLS 可能因 client hostname 與 upstream hostname 不同而失敗。
- Headless Service 設定 `clusterIP: None`，不配置 VIP 或 platform load balancing；DNS 直接回傳 Pod/endpoints 的 A/AAAA records，常用於 StatefulSet 或 client-side discovery。
- 無 selector 的 Service 可搭配自行管理的 EndpointSlice 整合 cluster 外 backend；不要將其誤解為任意 API-server proxy，`kubectl port-forward service/...` 對非 Pod endpoints 會被拒絕。

Readiness 很關鍵：Service 不判斷 business health，而是依 EndpointSlice condition 和 implementation 選擇可服務的 backends。`Pod Running`、`Ready`、出現在 EndpointSlice、以及 application transaction 成功是不同證據。

## Ingress、Ingress controller 與 Gateway API

| 問題 | Ingress | Gateway API |
| --- | --- | --- |
| API 狀態 | GA、已 frozen；不再新增功能，但沒有移除計畫 | add-on CRDs；核心 kinds 已有 stable resources，仍依 implementation 支援程度 |
| 主要模型 | 一個 Ingress resource 描述 HTTP/HTTPS host/path routing | `GatewayClass -> Gateway -> HTTPRoute/GRPCRoute` 等 role-oriented model |
| 職責分離 | 常依 annotations 混合 platform 與 application 設定 | infrastructure provider、cluster operator、application developer 邊界較清楚 |
| 表達能力 | 基本 host/path、TLS、default backend | protocol-aware routing、header match、weighted traffic、route attachment policy |
| 執行條件 | 必須安裝 Ingress controller | 必須安裝 Gateway API CRDs 及相容 controller |

建立 Ingress 或 Gateway resource 本身不會產生 data plane。Controller 需要 watch resource、配置 proxy/load balancer，並回寫 status。多個 Ingress controllers 應以 `ingressClassName` / IngressClass 明確選擇，避免 resource 被錯誤 controller 接管。

Ingress 的 `pathType` 應明確使用 `Exact` 或 `Prefix`；`ImplementationSpecific` 的行為取決於 controller。Ingress TLS 通常只保護 client 到 ingress point；若 backend hop 也需加密或驗證，必須另外設計 re-encryption、mTLS 或 service mesh。

新平台若 implementation 已支援，優先評估 Gateway API；既有 Ingress 不需要只因 frozen 就緊急移除。Migration 應驗證 controller feature parity、annotations、TLS、health checks、source IP、observability 與 rollback。

## EndpointSlice：Service backend 的可擴展來源

EndpointSlice 將一個 Service 的 backends 分割成多個 objects，避免單一 Endpoints object 在大型 Service 中成為更新與傳輸瓶頸。預設每個 slice 最多 100 endpoints；dual-stack Service 至少會依 IPv4/IPv6 address type 分開。

每個 endpoint 的重要資訊包括：

- `addresses`、`ports`、`nodeName`、`zone` 與 topology hints。
- `ready`：是否應接收一般 Service traffic。
- `serving`：endpoint 即使正在 terminating，application 是否仍能服務。
- `terminating`：endpoint 是否正在終止，可用於 connection draining 決策。

Legacy `Endpoints` API 自 Kubernetes 1.33 起 deprecated：它不支援 dual-stack、缺少 `trafficDistribution` 等新資訊，且大型 backend list 會截斷。新的 controller 或 troubleshooting 流程應以 `discovery.k8s.io/v1` EndpointSlice 為主。

## DNS：名稱發現，不是封包轉送

CoreDNS 等 cluster-aware DNS watches Services 並產生 records；kubelet 為 Pod 配置 resolver。一般 Service 的 FQDN 形式為：

```text
<service>.<namespace>.svc.<cluster-domain>
api.prod.svc.cluster.local
```

- 同 namespace 可使用短名稱 `api`；跨 namespace 至少使用 `api.prod`，production troubleshooting 最好直接測試 FQDN。
- 一般 Service 的 A/AAAA record 指向 ClusterIP；headless Service 指向一組 Pod/endpoint IPs。
- Named Service ports 會產生 SRV records，讓 client 同時發現 port 與 target name。
- `dnsPolicy` 未設定時預設是 `ClusterFirst`，不是名稱為 `Default` 的 policy；`hostNetwork` Pod 通常需要 `ClusterFirstWithHostNet`。
- 預設 search domains 與常見 `ndots:5` 會將短名稱或含少量 dots 的 query 依序展開，可能增加 DNS queries 與 latency。對外部服務可考慮完整 FQDN（常以 trailing dot 避免 search expansion），但要先驗證 application resolver 行為。

DNS 成功只證明 name resolution；不證明 EndpointSlice、NetworkPolicy、Service data plane、TLS 或 application 正常。

## NetworkPolicy：default allow 上的 additive allow rules

Kubernetes 預設 Pod ingress 與 egress 都是 non-isolated。只有支援 NetworkPolicy enforcement 的 CNI/network plugin 才會執行 policy；成功建立 YAML 但使用不支援的 plugin，實際上不會阻擋任何流量。

推薦做法是先在 namespace 建立 default-deny，再明確允許 DNS、必要 upstream、ingress controller 與監控路徑，並以真實 traffic tests 驗證：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

重要語意：

- ingress isolation 與 egress isolation 獨立；一條 Pod-to-Pod connection 必須同時被 source Pod 的 egress 及 destination Pod 的 ingress 允許。
- Policies 是 additive allow lists，沒有 rule priority 或 explicit deny；只要任一適用 policy 允許，該方向就允許。
- `podSelector` 只選同 namespace Pods；`namespaceSelector` 選 namespaces；同一個 `from`/`to` item 內同時放兩者是 AND，不同 list items 是 OR。YAML indentation 會改變安全語意。
- 標準 API 主要處理 TCP/UDP/SCTP 的 L3/L4 identity、CIDR 與 ports，不提供 HTTP path、method、user identity、TLS policy、流量 log 或 cluster-wide default policy。
- NAT 可能發生在 NetworkPolicy evaluation 之前或之後，因此 `ipBlock` 看到的 source/destination 可能是 client、node 或 load balancer IP；必須依 CNI、cloud 與 Service implementation 測試。
- `hostNetwork` traffic 通常會被視為 node traffic，跨 implementation 的 selector 行為有限；不能假設與一般 Pod 相同。

## Traffic policy、distribution 與 topology

這三種機制的語意不同：

| 機制 | 語意 | 找不到 local/close endpoint 時 |
| --- | --- | --- |
| `internalTrafficPolicy: Local` | 對 cluster-internal traffic 的嚴格 node-local restriction | 該 node 對此 Service 視為沒有 endpoints，可能直接失敗 |
| `externalTrafficPolicy: Local` | 外部流量只送 node-local endpoints，常用於保留 source IP/減少 hop | 沒有 local endpoint 的 node 不應承接該流量，需正確 LB health checks |
| `trafficDistribution` | `PreferSameNode`、`PreferSameZone` 等 routing preference | 可 fallback 到其他 endpoints；不是可用性保證 |
| Topology Aware Routing | EndpointSlice hints 優先 same-zone traffic | heuristic 無法平衡時回到 cluster-wide routing |

Topology Aware Routing 使用 `service.kubernetes.io/topology-mode: Auto`；1.27 前的名稱是 topology-aware hints，舊 annotation 不應用於新設定。它較適合流量在 zones 間大致均勻、每 zone 至少約三個 endpoints 的 Service；流量高度偏向單一 zone 時，可能壓垮該 zone 的 subset。它優化 latency、cross-zone cost 與 throughput，但不取代 topology spread、capacity planning 或 failover testing。

選擇原則：需要 hard locality 才使用 `Local`；只想降低 hop/cost 且仍要 fallback，使用 supported `trafficDistribution` preference；使用 topology hints 前先確認 kube-proxy/CNI implementation、endpoint 數與 zone traffic distribution。`ServiceTrafficDistribution` 與 `PreferSameTrafficDistribution` 已分別在 v1.33、v1.35 GA。

## IPv4/IPv6 dual-stack

Dual-stack 自 Kubernetes 1.23 為 stable，但「Kubernetes feature 可用」不代表 CNI、cloud load balancer、DNS、firewall、observability 與 application 全都支援。

- `ipFamilyPolicy: SingleStack`：只配置一個 address family。
- `PreferDualStack`：cluster 支援時配置兩個 families，否則 fallback 至 single-stack。
- `RequireDualStack`：必須同時配置 IPv4/IPv6，不支援時 Service 建立失敗。
- `ipFamilies` 決定 families 與 primary ordering；primary family 建立後不能直接改成另一 family。
- `.spec.clusterIPs` 可包含兩個 VIP；每個 EndpointSlice 只包含單一 address type。

Migration 需要檢查 Pod CIDRs、Service CIDRs、node addresses、CNI/IPAM、routes、security rules、load balancer target mode、AAAA records、client happy-eyeballs 行為，以及只支援 IPv4 的 dependencies。不要只因 Service 取得兩個 ClusterIPs 就宣告 migration 完成。

## ClusterIP allocation

Control plane 可從 Service CIDR 動態分配 ClusterIP，也允許在 CIDR 內指定靜態 IP；全 cluster 必須唯一。靜態地址通常只用於 cluster DNS 等真正需要 well-known IP 的基礎服務，applications 應優先使用 DNS name。

現代 allocator 將 Service CIDR 切出較低的 static subrange，動態配置優先從較高範圍開始，以降低 collision，但它不是 application 任意硬編碼 VIP 的理由。建立 cluster 時要估算 Services 數量並避免 Pod CIDR、node/VPC CIDR、on-prem network 與 Service CIDR 重疊；CIDR sizing 或 primary family 的事後變更通常比初始規劃昂貴。

## Troubleshooting：沿路徑找 failure domain

```text
name -> Service -> EndpointSlice -> policy -> route/proxy -> Pod -> application
```

1. **DNS**：在相同 namespace/client Pod 查短名稱與 FQDN；檢查 `/etc/resolv.conf`、CoreDNS Pods/logs 與 NetworkPolicy 是否允許 UDP/TCP 53。
2. **Service contract**：確認 selector、`port`/`targetPort`、type、IP families 與 traffic policies。
3. **Backend discovery**：檢查 EndpointSlices 是否存在、addresses/ports 是否正確，以及 `ready`、`serving`、`terminating` conditions。
4. **Direct backend**：從 debug Pod 測 Pod IP:targetPort；若失敗，先查 application、listen address、CNI route 與 policy，而不是怪 Service。
5. **Service VIP**：Pod IP 成功但 ClusterIP 失敗時，查 kube-proxy/eBPF rules、Service implementation 與 node-specific behavior。
6. **Edge**：查 IngressClass/GatewayClass、controller status/events/logs、LB health checks、TLS/SNI、route attachment 與 backend reference。
7. **Scope comparison**：從同 node、跨 node、同 zone、跨 zone、cluster 外各測一次，定位 topology、SNAT 或 traffic policy 問題。

## 常見面試題的精準回答

### Service 如何找到 Pods？

Service selector 由 controller 轉成 EndpointSlices；Service data plane watches/consumes EndpointSlices，將 traffic 導向 eligible endpoints。Service 不是每次收到 packet 才即時查 labels，也不是 Service object 自己轉送流量。

### ClusterIP 是真的綁在某張 network interface 上嗎？

通常不是。它是由 kube-proxy、eBPF 或其他 implementation 以 packet-processing rules 實現的 virtual IP。實際細節依 dataplane 而異，因此回答應著重 API contract，不把 iptables 當成唯一實作。

### Ingress 和 Gateway API 怎麼選？

Ingress 是穩定但 frozen 的 HTTP/HTTPS API，既有環境仍可運作；Gateway API 將 infrastructure、listener 與 application routes 分離，支援更豐富且較 portable 的 routing。新設計在 controller 成熟且功能相容時優先評估 Gateway API，既有系統則依 migration benefit 與 risk 決定。

### NetworkPolicy 為什麼建立成功卻沒有擋流量？

API server 只儲存 resource；必須由支援 NetworkPolicy 的 CNI/policy engine enforcement。接著確認 policy 是否選到 Pods、方向是否包含 Ingress/Egress、其他 additive policy 是否允許，以及 NAT/hostNetwork 是否改變匹配結果。

### `internalTrafficPolicy: Local` 和 topology-aware preference 有何差別？

`Local` 是 hard restriction：node 沒有 local endpoint 就沒有可用 backend。Topology-aware/`trafficDistribution` 是 preference，可在 local endpoint 不合適時 fallback。前者用 availability 換 locality guarantee，後者偏向 optimization。

### DNS 正常為什麼 Service 仍無法連線？

DNS 只把名稱解析為 ClusterIP 或 endpoint IP。還要分別驗證 EndpointSlice readiness、Service port mapping、dataplane rules、NetworkPolicy、TLS 與 application listener。

## 版本與淘汰提醒

- Ingress API 是 **frozen，不是 deprecated**；目前沒有移除計畫，但新功能轉向 Gateway API。
- Legacy `Endpoints` API 自 Kubernetes 1.33 **deprecated**；使用 EndpointSlice。
- `service.kubernetes.io/topology-aware-hints` 是 1.27 前的舊 annotation；目前文件使用 `service.kubernetes.io/topology-mode: Auto`，並可評估 `trafficDistribution`。
- Service `.spec.externalIPs` 在目前 Kubernetes 1.36 文件標為 **deprecated**；新架構應使用 external load balancer controller 或 Gateway API implementation。
- Feature state、controller 支援與 managed Kubernetes 行為會隨版本不同；上線前以目標 cluster version、CNI 與 controller documentation 驗證。

## 參考資料

- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [IPv4/IPv6 dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/)
- [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Service ClusterIP allocation](https://kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/)
- [Service Internal Traffic Policy](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)

---

# Kubernetes Networking: Interview-Depth Guide

Kubernetes networking is a set of contracts rather than one proxy. CNI supplies Pod connectivity; Services provide stable virtual entry points; EndpointSlices publish real backends; DNS supplies discovery; Ingress or Gateway API handles north-south routing; and NetworkPolicy constrains L3/L4 traffic. The data plane may be implemented by kube-proxy, eBPF, cloud load balancers, or other controllers.

## End-to-end model

For east-west traffic, a client resolves a Service name, connects to its ClusterIP, and the Service data plane chooses an eligible endpoint from EndpointSlices. For north-south traffic, an edge load balancer or proxy configured by an Ingress/Gateway controller forwards through a Service to a ready backend.

Separate control plane from data plane in interview answers. API resources express desired state; controllers reconcile infrastructure and status; proxies, CNI/eBPF programs, and load balancers actually forward or reject packets. Resource creation alone does not prove reachability.

## Service and EndpointSlice

- `port` is the client-facing Service port; `targetPort` is the backend port.
- `ClusterIP` is the default internal virtual IP. `NodePort` exposes a port on each node. `LoadBalancer` requires an external implementation. `ExternalName` is only a DNS mapping and can cause HTTP/TLS hostname issues.
- A headless Service uses `clusterIP: None`; it has no VIP or platform load balancing, and DNS returns endpoint addresses directly.
- Services without selectors can target manually managed EndpointSlices, but API-server proxying to non-Pod endpoints is deliberately restricted.
- EndpointSlice is the scalable source of backend addresses and includes `ready`, `serving`, and `terminating` conditions plus node, zone, and address-family data.
- The legacy Endpoints API is deprecated as of Kubernetes 1.33; it lacks dual-stack and modern traffic metadata and can truncate large endpoint sets.

Do not equate `Running`, `Ready`, EndpointSlice membership, and a successful application transaction. They prove different stages of the serving path.

## Ingress and Gateway API

Ingress is GA and frozen, not scheduled for removal. It handles basic HTTP/HTTPS host and path routing, but only when an Ingress controller is installed. Use `ingressClassName`, explicit path types, and controller-specific validation. TLS termination at ingress does not automatically encrypt the backend hop.

Gateway API is an add-on based on CRDs and a compatible controller. Its stable resource model separates responsibilities through `GatewayClass`, `Gateway`, and route resources such as `HTTPRoute` and `GRPCRoute`. It supports portable, protocol-aware matching, weighting, and attachment policy that often required Ingress annotations.

Prefer evaluating Gateway API for new platforms when the implementation is mature. Existing Ingress deployments do not need emergency replacement merely because the API is frozen; migrate based on feature parity, operational value, testing, and rollback readiness.

## DNS

- A Service FQDN is `<service>.<namespace>.svc.<cluster-domain>`.
- Short names resolve within the caller's namespace; qualify cross-namespace names.
- A regular Service record resolves to its ClusterIP; a headless Service record resolves to endpoint IPs. Named ports also produce SRV records.
- The default Pod DNS policy is `ClusterFirst`; a `hostNetwork` Pod normally needs `ClusterFirstWithHostNet`.
- Search domains and `ndots` can amplify queries and latency. Validate resolver behavior before changing them.

Successful DNS resolution proves discovery only—not endpoint readiness, policy, TLS, forwarding, or application health.

## NetworkPolicy

Pods are ingress- and egress-non-isolated by default. A NetworkPolicy has no enforcement effect unless the installed network implementation supports it. Begin with namespace default-deny policies, then explicitly allow DNS and required application and operational flows, validating with real connectivity tests.

Ingress and egress isolation are independent. For Pod-to-Pod traffic, source egress and destination ingress must both allow the connection. Policies combine additively; the standard API has no priority or explicit deny. A `podSelector` is namespace-local; a combined `namespaceSelector` and `podSelector` in one list item is AND, while separate items are OR.

Standard NetworkPolicy is primarily L3/L4, not HTTP authorization, WAF, traffic logging, or a cluster-wide policy system. NAT and `hostNetwork` can change the addresses visible to enforcement, so behavior must be tested against the actual CNI, cloud, and Service implementation.

## Locality and traffic distribution

- `internalTrafficPolicy: Local` is a hard node-local restriction. With no local endpoint, the Service has no usable backend on that node.
- `externalTrafficPolicy: Local` restricts external traffic to node-local endpoints and is often used to preserve source IP; load-balancer health checks must exclude nodes without local endpoints.
- `trafficDistribution` expresses preferences such as `PreferSameNode` or `PreferSameZone` and permits fallback.
- Topology Aware Routing uses EndpointSlice hints and `service.kubernetes.io/topology-mode: Auto`. It works best with balanced zone traffic and roughly three or more endpoints per zone; otherwise it may fall back to cluster-wide routing.

Use strict `Local` semantics only when locality is a requirement. Use distribution preferences for cost or latency optimization where fallback is safer. `ServiceTrafficDistribution` and `PreferSameTrafficDistribution` reached GA in v1.33 and v1.35 respectively. Neither replaces capacity, topology spread, or failure testing.

## Dual-stack and address planning

Dual-stack is stable, but the CNI, load balancers, firewall, DNS, monitoring, clients, and dependencies must all support it.

- `SingleStack` allocates one family.
- `PreferDualStack` allocates both when available and falls back otherwise.
- `RequireDualStack` fails Service creation if both families are unavailable.
- `ipFamilies` selects family order; the primary family cannot simply be changed after creation.
- `.spec.clusterIPs` can hold two VIPs, while each EndpointSlice uses one address type.

ClusterIPs can be dynamically assigned or statically selected from the Service CIDR, and must be unique cluster-wide. Reserve static IPs for infrastructure that truly needs a well-known address, such as cluster DNS; applications should use DNS. Plan non-overlapping Pod, Service, node/VPC, and on-premises CIDRs before cluster creation.

## Troubleshooting sequence

Follow the serving path rather than restarting components at random:

1. Resolve the short name and FQDN from the actual client namespace; inspect Pod resolver config and DNS policy.
2. Check the Service selector, ports, type, IP families, and traffic policies.
3. Inspect EndpointSlice addresses, ports, and readiness/termination conditions.
4. Test the Pod IP and target port directly from a debug Pod.
5. If Pod IP works but ClusterIP fails, inspect kube-proxy/eBPF state and node-specific behavior.
6. Inspect IngressClass/GatewayClass, controller status/events/logs, TLS/SNI, load-balancer health, and route attachment.
7. Compare same-node, cross-node, same-zone, cross-zone, and external tests to expose topology, SNAT, or locality-policy failures.

## Interview checkpoints

- A Service selector is reconciled into EndpointSlices; the data plane consumes them. The Service object does not forward packets itself.
- A ClusterIP is usually a virtual address implemented by packet-processing rules, not an IP bound to one interface. Do not describe iptables as the only possible implementation.
- An accepted NetworkPolicy object without a supporting enforcement engine changes no traffic.
- `Local` traffic policy is a hard restriction; topology-aware distribution is a preference with fallback.
- DNS success does not prove the complete serving path.

## Version notes

- Ingress is **frozen, not deprecated**, and has no removal plan; new feature development is centered on Gateway API.
- The legacy Endpoints API is **deprecated since Kubernetes 1.33**; use EndpointSlice.
- The pre-1.27 topology-aware-hints annotation is old; current documentation uses `service.kubernetes.io/topology-mode: Auto` and also documents `trafficDistribution`.
- Service `.spec.externalIPs` is **deprecated in the current Kubernetes 1.36 documentation**; prefer an external load-balancer controller or Gateway API implementation for new designs.
- Verify feature state and behavior against the target Kubernetes version, CNI, cloud provider, and controller before production rollout.

## References

- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [IPv4/IPv6 dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/)
- [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Service ClusterIP allocation](https://kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/)
- [Service Internal Traffic Policy](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
