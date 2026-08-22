# Problem and Success Criteria

## Customer problem

A synchronous report-processing path couples upload latency to processing
capacity and downstream availability. Bursts create timeouts or manual recovery,
while duplicate delivery and partial failure are difficult to audit.

The proposed modernization introduces an asynchronous boundary without changing
the business meaning of a report.

## Scope

In scope are object intake, validation, queueing, processing, idempotency,
failure isolation, replay, telemetry, least-privilege access, and a reversible
cutover. Changes to the report's business rules or downstream data model require
separate approval.

## Acceptance criteria

| Criterion | Required test |
|---|---|
| Correctness | A valid object produces the expected result exactly once at the business level |
| Duplicate safety | Re-delivering the same object/version does not duplicate side effects |
| Failure isolation | A poison message reaches the DLQ without blocking healthy work |
| Burst tolerance | Queue age and drain time remain within agreed objectives under test load |
| Downstream protection | Concurrency and retry behavior remain within dependency capacity |
| Traceability | An operator can correlate object, event, Lambda invocation, result, and failure |
| Recovery | Reviewed redrive and rollback exercises complete successfully |
| Security | IAM, encryption, logging, retention, and sensitive-data controls pass review |

No criterion is marked achieved until test output and an accountable owner are
recorded.
