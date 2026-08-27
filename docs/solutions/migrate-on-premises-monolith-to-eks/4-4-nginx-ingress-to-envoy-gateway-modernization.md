# NGINX Ingress to Envoy Gateway Modernization

> **Phase:** 4 Validate · **Category:** Modernization
> **Priority topic:** 23
> **Evidence standard:** Gateway API and Envoy Gateway mechanisms are supported
> by retained architecture and operational evidence. Each migration still
> requires route, policy, traffic, and legacy-load-balancer validation.

## Abstract

Gateway API is the Kubernetes routing contract. Envoy Gateway is the controller
and control plane that implements it. Envoy Proxy is the generated data plane
that handles live traffic. These roles are not interchangeable.

## Translation map

| NGINX-era concern | Target object or control | Parity evidence |
|---|---|---|
| Ingress class and controller configuration | `GatewayClass` and `EnvoyProxy` | Accepted class and healthy controller/data plane |
| TLS listener and host | `Gateway` listener and certificate reference | TLS chain, hostname, renewal, and rotation test |
| Host/path routing | `HTTPRoute` rules and backend references | `Accepted=True`, `ResolvedRefs=True`, and request tests |
| Rewrites, headers, redirects | Gateway API filters or typed Envoy policy | Golden request/response comparison |
| Authentication and authorization | `SecurityPolicy` or approved external authorization | Allowed, denied, timeout, and failure-mode tests |
| Rate, timeout, retry, and circuit controls | Envoy Gateway traffic policies | Steady, burst, failure, and recovery tests |
| External load balancer | Generated Service plus AWS load-balancer integration | DNS, listener, target, security-group, and traffic evidence |

## Implementation sequence

1. Inventory every Ingress, annotation, class, controller ConfigMap,
   certificate, DNS record, external load balancer, client pinned to an ELB
   hostname, and policy behavior.
2. Classify each annotation as portable Gateway API, Envoy-specific policy,
   application responsibility, or unsupported behavior requiring redesign.
3. Deploy GatewayClass, Gateway, routes, policies, and telemetry without moving
   production DNS.
4. Run direct endpoint tests for routing, TLS, CORS, authentication, limits,
   uploads, streaming, WebSockets/gRPC, errors, and graceful drain as
   applicable.
5. Shift a bounded hostname, route, or DNS weight and observe both data planes.
6. Move canonical DNS only after parity and rollback pass.
7. Remove legacy Ingress sources, controller, LoadBalancer Service, RBAC, class,
   and cloud load balancer in ownership order.

## Rollback and retirement gates

- rollback restores the prior canonical route without changing application data;
- the legacy target is retained only for the approved observation window;
- no legitimate traffic reaches the old load balancer before deletion;
- deleting a controller Deployment alone is not accepted as retirement because
  the LoadBalancer Service and cloud load balancer can remain;
- internal application proxies are distinguished from the obsolete Kubernetes
  ingress controller;
- generated and controller-owned resources are not manually edited as source.

## References

- [Envoy Gateway: Interview-Depth Guide](../../practices/k8s/envoy-gateway.md)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway documentation](https://gateway.envoyproxy.io/)
