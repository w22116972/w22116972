# Observability and SRE Operating Model

> **Phase:** 5 Operate · **Category:** Modernization
> **Priority topic:** 19
> **Evidence standard:** Metrics, logs, traces, dashboards, alert, and runbook
> patterns are retained. No universal SLO or incident-improvement percentage is
> claimed.

## Abstract

Observability begins with customer and business outcomes, then correlates them
with application, dependency, Kubernetes, node, and AWS signals. Collecting more
telemetry without a decision or owner adds cost without improving operations.

## Signal hierarchy

| Layer | Core questions | Example signals |
|---|---|---|
| Business | Is the user outcome succeeding? | Successful transactions, jobs, tenants, freshness, business backlog |
| Service | Is the workload meeting its objective? | Availability, p50/p95/p99 latency, errors, saturation |
| Dependency | Which downstream path limits the outcome? | Timeouts, retries, connection pools, queue depth, database latency |
| Kubernetes | Is orchestration contributing to impact? | Readiness, restarts, throttling, OOM, pending, eviction, rollout state |
| Infrastructure | Is AWS capacity or networking limiting service? | Node, volume, load balancer, quota, subnet IP, DNS and control-plane signals |
| Delivery | Did a change cause the impact? | Source, image digest, chart, GitOps revision, config and feature flag |

## Architecture

Use CloudWatch and/or Prometheus-compatible metrics, centralized structured
logs, and OpenTelemetry-compatible tracing according to ownership, retention,
query, portability, and cost requirements. Propagate correlation identifiers
without recording credentials or unnecessarily sensitive payloads.

## Implementation sequence

1. Define SLIs, SLOs, error budgets, owners, and user-impact thresholds.
2. Establish consistent service, environment, version, route, tenant/cohort, and
   dependency dimensions with controlled cardinality.
3. Instrument the real request and asynchronous paths across legacy and EKS
   components.
4. Create audience-specific dashboards: executive/customer outcome, service,
   platform, migration wave, and incident deep dive.
5. Alert on actionable symptoms and objective burn, not every component event.
6. Link alerts to runbooks and test notification, escalation, missing telemetry,
   and incident communication.
7. Tune sampling and retention from diagnostic and compliance needs.

## Acceptance and rollback

- a synthetic or controlled failure appears in metrics, logs, traces, alerting,
  and the correct runbook;
- operators can trace user impact through route, service, dependency, and
  release;
- telemetry loss is detectable and blocks unsafe cutover when required;
- high-cardinality or sensitive fields are rejected or transformed;
- cost and retention have owners and budgets;
- instrumentation can be disabled or sampled down without redeploying unrelated
  services.

## References

- [Amazon EKS observability and cost guidance](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-observability.html)
- [AWS Observability and Operational Automation](../../practices/aws/observability-operational-automation.md)
