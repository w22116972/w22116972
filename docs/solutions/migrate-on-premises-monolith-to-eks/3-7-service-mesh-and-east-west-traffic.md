# Service Mesh and East-West Traffic Modernization

> **Phase:** 3 Execute · **Category:** Modernization
> **Priority topic:** 31
> **Evidence standard:** Optional architecture decision. No service-mesh
> deployment or production benefit is claimed by this case study.

## Abstract

A service mesh is justified by cross-service security, traffic, and telemetry
requirements—not by the number of microservices alone. NetworkPolicy remains
the simpler baseline for Layer 3 and Layer 4 isolation.

## Key decisions

| Requirement | Start with | Add a mesh when |
|---|---|---|
| Namespace and workload isolation | Kubernetes NetworkPolicy | Layer 7 identity or traffic policy is required |
| External north-south routing | Gateway API and Envoy Gateway | East-west controls must share mesh identity or policy |
| Service authentication | Workload identity and application TLS | Uniform service-to-service mTLS and rotation are required |
| Retries, timeouts, circuit breaking | Application libraries and Gateway policy | Cross-language consistency outweighs proxy and control-plane cost |
| Distributed telemetry | OpenTelemetry instrumentation | Transparent service traffic coverage is required and sampled responsibly |

## Implementation sequence

1. Record the exact security, traffic-management, or observability gap that the
   mesh must close.
2. Compare sidecar, ambient, library, and gateway-only approaches for workload
   compatibility, latency, resources, failure modes, and operations.
3. Pilot one non-critical service path with explicit success and removal
   criteria.
4. Establish certificate authority, identity, trust-domain, policy ownership,
   proxy resources, upgrade, and emergency-bypass procedures.
5. Test mTLS, authorization, retries, timeouts, circuit breaking, topology,
   telemetry, control-plane loss, and certificate rotation.
6. Expand only when measured benefit exceeds added latency, compute cost, and
   incident complexity.

## Guardrails

- retry multiplication across application, mesh, gateway, and client is bounded;
- timeouts decrease toward downstream dependencies and fit the user objective;
- mesh denial and certificate failure are distinguishable from application
  failure in telemetry;
- policies are reviewed as code and denied flows are tested;
- sidecar or node-level proxy resources are included in scheduling and cost;
- rollback removes traffic interception without breaking service discovery or
  workload identity.

## References

- [Amazon EKS network security: service mesh or NetworkPolicy](https://docs.aws.amazon.com/eks/latest/best-practices/network-security.html)
- [Envoy Gateway](../../practices/k8s/envoy-gateway.md)
