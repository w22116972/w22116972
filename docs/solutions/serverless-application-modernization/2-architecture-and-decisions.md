# Architecture and Decisions

## Abstract

This phase defines the event contract carrying bucket, immutable object
version, business document type, schema version, creation time, and
correlation identifier, and derives the idempotency key from object version
plus business identity. It records processing behavior, capacity and failure
decisions, and the alternatives considered.

## Event contract

The event identifies the bucket, immutable object version, business document
type, schema version, creation time, and correlation identifier. It contains no
unnecessary sensitive payload. The object version plus business identity forms
the idempotency key.

## Processing behavior

1. Amazon S3 persists the source and emits an event.
2. Amazon SQS buffers events and defines retry visibility.
3. Lambda validates schema and authorization context.
4. The processor claims or checks the idempotency key.
5. It writes the business result and completion record atomically where
   feasible.
6. Repeated failures move to a DLQ with diagnostic context.

## Capacity and failure decisions

- Queue visibility timeout exceeds the expected function duration and is tuned
  through load tests.
- Reserved concurrency protects downstream systems; alarms detect growing event
  age before the business objective is missed.
- Maximum receive count prevents endless retries.
- Replay is an operator-owned action that first confirms the defect or
  dependency failure has been corrected.
- Object lifecycle and queue retention are long enough for the agreed recovery
  window.

## Alternatives

Direct S3-to-Lambda invocation is simpler but offers less buffering and redrive
control. Amazon EventBridge is preferred when routing rules, multiple consumers,
or cross-account event integration are primary. AWS Step Functions is added
only when the business process needs explicit multi-step orchestration.
