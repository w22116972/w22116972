# Serverless and Event-Driven Extraction

> **Phase:** 3 Execute · **Category:** Modernization
> **Priority topic:** 25
> **Evidence standard:** Target architecture and decision pattern. No completed
> production serverless modernization outcome is claimed.

## Abstract

EKS is not the default destination for every extracted capability. Event-driven
or short-lived functions can reduce platform ownership when their execution,
latency, state, networking, and cost characteristics fit managed services.

## Placement decision

| Workload characteristic | Candidate | Reconsider when |
|---|---|---|
| Long-running service, Kubernetes policy, specialized runtime, or portable control plane | Amazon EKS | Platform complexity exceeds the workload value |
| Event-triggered, stateless, bounded execution with variable demand | AWS Lambda | Duration, startup, concurrency, package, networking, or unit cost is unsuitable |
| Durable asynchronous integration | Amazon SQS, SNS, or EventBridge | Ordering, throughput, delivery semantics, payload, or replay requirements do not fit |
| Multi-step orchestration with explicit state | AWS Step Functions | Workflow volume, latency, or service integration does not fit |
| Managed container without Kubernetes requirements | Amazon ECS or AWS Fargate | EKS-specific controls or ecosystem are required |

## Extraction pattern

```text
monolith transaction
  -> transactional outbox or approved event source
    -> versioned event contract
      -> queue or event bus
        -> idempotent consumer
          -> domain-owned result and observable failure path
```

## Implementation sequence

1. Select a bounded capability with clear trigger, owner, data contract, and
   measurable result.
2. Define delivery semantics, ordering, deduplication, retry budget, timeout,
   dead-letter handling, retention, replay, and schema compatibility.
3. Prevent dual-write inconsistency with an outbox, change stream, or another
   explicitly validated pattern.
4. Implement least-privilege identities and encrypt events and state.
5. Load-test steady, burst, poison-message, dependency-failure, replay, and
   throttling behavior.
6. Route one event class or cohort to the new consumer while retaining an
   observable fallback.
7. Retire the legacy handler only after backlog, duplicate, ordering, and data
   reconciliation pass.

## Acceptance and rollback

- producer and consumer contract tests cover old and new schema versions;
- duplicate delivery does not duplicate the business outcome;
- poison messages are recoverable and do not block unrelated work;
- concurrency is bounded to protect databases and downstream APIs;
- replay is authorized, rate-limited, observable, and reconciled;
- rollback stops new routing without discarding accepted events.

## References

- [Serverless Application Modernization](../serverless-application-modernization/README.md)
- [Serverless and Event-Driven Architecture](../../practices/aws/serverless-event-driven-architecture.md)
