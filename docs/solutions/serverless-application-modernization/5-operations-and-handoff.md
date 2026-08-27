# Operations and Handoff

## Abstract

This phase defines the operational signals an operator watches, the runbooks
that respond to them, and what must be true before handoff is accepted.

## Operational signals

Operators monitor accepted object count, successful business outcomes, Lambda
errors and throttles, queue depth, age of oldest message, receive count, DLQ
growth, concurrency, duration, and downstream failures. Alerts identify an
owner, impact, dashboard, runbook, and escalation path.

## Runbooks

- investigate a growing backlog and distinguish demand, throttling, code, and
  downstream causes;
- inspect and quarantine a poison message without exposing sensitive data;
- correct the cause and perform a bounded DLQ redrive;
- lower concurrency during downstream instability;
- roll back the Lambda alias or producer release;
- reconcile source objects and business outcomes after a partial outage.

## Exit criteria

The customer receives architecture decisions, IaC and application ownership,
event contracts, dashboards, alerts, runbooks, access paths, cost controls,
security and retention decisions, test evidence, and residual risks.

Handoff is complete only after customer operators diagnose a test backlog,
redrive a controlled failure, trace one transaction, and execute rollback
without relying on the delivery team.
