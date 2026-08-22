# Amazon EKS Best Practices：Networking

EKS networking 必須提供可路由的 pod addresses、具韌性的 control-plane 與 service connectivity、明確的 traffic isolation，以及足夠的 address space 供成長使用。應在 cluster creation 前規劃 network，因後續 CIDR 與 subnet 變更可能造成 disruption。

> 最後檢視：2026-08-20

## VPC、pod traffic 與 load balancing

- 使用多個 Availability Zones，將 worker nodes 放在 private subnets，只公開需要 public reachability 的 load balancers 或 endpoints。
- 有意識地選擇 public、private 或 combined cluster endpoint access；限制 public endpoint CIDRs，並驗證 nodes 與 administrators 的 redundant private connectivity。
- 為 steady state、rollouts、node replacement 與 burst scaling 規劃 subnets；在耗盡前監控 available addresses 與 VPC CNI allocation。
  - Rolling update 期間 old/new nodes 與 pods 會短暫並存，因此只按 steady-state 數量配置 IP，可能在 deployment 或 recovery 時耗盡。
- 新 clusters 在周邊 systems 支援時優先考慮 IPv6；IPv4 則考慮 prefix delegation、warm IP/prefix targets 與 subnet CIDR reservations。
  - Prefix delegation 可提高每個 node 的 pod density；warm targets 以預留 IP 換取較快 pod startup，兩者都需納入 subnet capacity 計算。
- 使用 Amazon VPC CNI managed add-on、dedicated least-privileged IAM role、NetworkPolicies 的 default-deny ingress/egress 並明確允許 DNS；依需求結合 security groups for pods 與 network policies。
  - NetworkPolicies 管理 pod/namespace flows，security groups for pods 管理 AWS/VPC resource-level flows；兩者作用層次不同，不能互相完全取代。
  - Standard EKS 的 VPC CNI NetworkPolicy enforcement 預設未啟用；須使用支援的 VPC CNI version 並設定 `ENABLE_NETWORK_POLICY=true`，否則建立 policy 不代表 traffic 已受限制。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/network-security.html)
  - `NETWORK_POLICY_ENFORCING_MODE=strict` 可讓新 pod 在 policies 完成設定前先採 default-deny；standard mode 在初始化期間會短暫 default-allow，因此應依 startup availability 與 isolation requirement 選擇。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy-configure.html)
- 依 source NAT、node access、service routing 與 compatibility 選擇 security-group enforcement mode，並監控 CNI、DNS、conntrack、packet errors 與 interface saturation。
  - 同時使用 VPC CNI NetworkPolicy 與 security groups for pods 時，SGP 必須使用 `POD_SECURITY_GROUP_ENFORCING_MODE=standard`；這個設定不同於前述 `NETWORK_POLICY_ENFORCING_MODE`，不可混用。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/network-security.html)
- HTTP/HTTPS 使用 Application Load Balancer；TCP、source-IP preservation、static-address 或 non-DNS needs 使用 Network Load Balancer，並由 AWS Load Balancer Controller provision。
- 將 load-balancer health checks 對齊 readiness，使用 readiness gates；shutdown 時 fail readiness、停止新工作、drain connections、完成 in-flight requests，並設定足夠的 `terminationGracePeriodSeconds`。
  - 若 pod 已終止但仍被 load balancer 視為 healthy，traffic 會送到失效 target；若 grace period 太短，尚未完成的 requests 會被強制中斷。

## Connectivity、availability 與 lifecycle

- private subnets 需要 resilient internet egress 時，按 Availability Zone 部署 NAT Gateways，或用 VPC endpoints 移除不必要的 NAT dependencies。
  - Cross-AZ 使用單一 NAT Gateway 會增加 data-transfer cost，且該 AZ 或 NAT 故障時可能同時切斷多個 zones 的 egress。
- 依 routing、scale、transitivity 與 ownership 選擇 peering、Transit Gateway 或其他 inter-VPC connectivity。
- 在 scale-out、deployment 與 node drain 期間驗證 DNS resolution、service routing、source IP、network policy 與 load-balancer registration。
- 使用 PDBs 與 topology distribution，避免 deployments、drains 與 zonal events 一次移除過多 backends。

## Review checklist

- CIDRs、subnets、IP allocation、endpoint access 與 Availability Zones 有 growth headroom。
- Network policy、security groups、DNS、load balancing 與 NAT 行為已在 lifecycle events 中測試。
- Pod readiness、connection draining 與 graceful termination 已與 traffic routing 對齊。
- Cross-zone、cross-VPC、internet egress 與 AWS service paths 有明確 ownership 與監控。

---
# Amazon EKS Best Practices: Networking

EKS networking must provide routable pod addresses, resilient control-plane and
service connectivity, deliberate traffic isolation, and enough address space for
growth. Plan the network before cluster creation because later CIDR and subnet
changes can be disruptive.

## VPC and subnet design

- Use multiple Availability Zones. Place worker nodes in private subnets and
  expose only the load balancers or endpoints that require public reachability.
- Select public, private, or combined cluster endpoint access intentionally.
  Restrict public endpoint CIDRs and verify redundant private connectivity for
  nodes and administrators.
- Size subnets for steady state, rollouts, node replacement, and burst scaling.
  Monitor available addresses and VPC CNI address allocation before exhaustion.
- Prefer IPv6 for new clusters when the surrounding systems support it. For
  IPv4 clusters, consider prefix delegation, tune warm IP or prefix targets, and
  use subnet CIDR reservations to limit fragmentation.
- Use custom networking only when it solves a defined address or isolation
  requirement. Automate per-zone configuration, recalculate maximum pods, and
  replace existing pods after enabling secondary networking.

## VPC CNI and pod traffic

- Use the Amazon VPC CNI managed add-on, give it a dedicated least-privileged IAM
  role, back up custom settings, and test upgrades regularly.
- Start Kubernetes network policy with default-deny ingress and egress, allow
  DNS explicitly, and add workload flows incrementally.
- Use security groups for pods for AWS-aligned, resource-level controls. Use
  network policies for pod and namespace segmentation; combine them when both
  layers are required.
- Select standard or strict security-group enforcement mode from the required
  source NAT, node access, service routing, and compatibility behavior. Test
  probes and graceful termination.
- Monitor CNI allocation, network-policy enforcement, DNS latency and
  throttling, conntrack pressure, packet errors, and interface saturation.

## Load balancing and lifecycle

- Use an Application Load Balancer for HTTP/HTTPS routing and a Network Load
  Balancer for TCP, source-IP preservation, static-address, or non-DNS needs.
- Provision load balancers with the AWS Load Balancer Controller. Prefer IP
  targets when direct pod routing and faster registration are appropriate.
- Align load-balancer health checks with Kubernetes readiness. Use pod readiness
  gates so a new pod is not considered ready before its target is healthy.
- Implement graceful shutdown: fail readiness, stop accepting new work, drain
  connections, allow in-flight requests to finish, and set sufficient
  `terminationGracePeriodSeconds`.
- Use PDBs and topology distribution so deployments, drains, and zonal events do
  not remove too many backends at once.

## Connectivity and availability

- Deploy NAT Gateways per Availability Zone when private subnets require
  resilient internet egress, or use VPC endpoints to remove unnecessary NAT
  dependencies.
- Choose peering, Transit Gateway, or other inter-VPC connectivity from routing,
  scale, transitivity, and ownership requirements.
- Validate DNS resolution, service routing, source IP behavior, network policy,
  and load-balancer registration during scale-out, deployment, and node drain.

## Review checklist

- Subnets have tested headroom for burst scaling and rolling replacement.
- Nodes and operators retain redundant access to the private cluster endpoint.
- CNI, network policies, pod security groups, DNS, and load balancers are monitored.
- Health checks, readiness gates, termination, and disruption budgets work together.
- Network design and routing changes are tested before production rollout.
