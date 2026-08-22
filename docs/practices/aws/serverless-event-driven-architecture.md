# Serverless and Event-Driven Architecture

## Purpose

Use managed events, queues, functions, and workflows to decouple producers from
consumers while making delivery semantics, retries, failure handling, and
capacity explicit.

## Practices

- Define the event contract before choosing services.
  - Record the producer, event meaning, schema, partition or ordering needs,
    retention, sensitivity, and expected volume.
  - Version schemas compatibly and give consumers time to migrate.
- Select the integration pattern intentionally.
  - Use Amazon SQS to buffer work and absorb demand spikes.
  - Use Amazon SNS for fan-out and Amazon EventBridge for event routing across
    decoupled producers and consumers.
  - Use AWS Step Functions when business steps, branching, waiting, or
    compensation require visible orchestration.
- Make consumers idempotent.
  - Assume an event can be delivered more than once and use a stable business
    key or event identifier to prevent duplicate side effects.
  - Keep functions stateless and store durable state in an appropriate managed
    data service.
- Bound retries and failure paths.
  - Configure visibility timeout, exponential backoff, maximum attempts, and a
    dead-letter or failure destination.
  - Alarm on event age, queue depth, throttling, iterator lag, errors, and
    dead-letter growth.
- Protect downstream systems.
  - Set reserved concurrency, batch size, and rate controls according to
    database, API, and partner-service capacity.
  - Use partial batch responses where supported so successful records are not
    retried with failed records.
- Propagate context without leaking data.
  - Carry correlation identifiers and business-safe metadata through the event
    path.
  - Apply least-privilege IAM, encryption, and data-retention controls at every
    hop.

## Acceptance tests

- Duplicate delivery produces one business outcome.
- A poison message reaches a visible failure path without blocking the queue.
- A downstream outage does not create unbounded concurrency or retry storms.
- Backlog recovery meets the stated recovery-time objective.
- Operators can trace an event from producer through router to destination.

## References

- AWS Serverless Applications Lens, [Event-driven architectures](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/event-driven-architectures.html)
- AWS Lambda Developer Guide, [Best practices for working with AWS Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- AWS Lambda Developer Guide, [Understanding application design](https://docs.aws.amazon.com/lambda/latest/dg/concepts-application-design.html)
