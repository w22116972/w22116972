# AWS Observability and Operational Automation

## Purpose

Connect technical telemetry to customer and business outcomes, then use safe
automation to reduce detection and recovery time. Observability is an operating
capability: dashboards without owners, thresholds, and response procedures are
not enough.

## Practices

- Start from service objectives and failure modes.
  - Define availability, latency, correctness, throughput, durability, and cost
    signals that reflect user impact.
  - Track error-budget consumption or an equivalent risk measure so alerts lead
    to decisions.
- Correlate metrics, logs, traces, events, and changes.
  - Propagate request and correlation identifiers across service boundaries.
  - Record deployment, configuration, scaling, and infrastructure events beside
    runtime telemetry.
- Build actionable alerts.
  - Every page identifies impact, severity, owner, dashboard, runbook, and
    escalation path.
  - Prefer symptom and objective alerts over noisy low-level conditions.
- Use the appropriate AWS telemetry sources.
  - Combine Amazon CloudWatch metrics and logs, AWS X-Ray traces where
    applicable, AWS CloudTrail activity, AWS Config history, and VPC Flow Logs
    according to the investigation need.
  - Protect sensitive data through filtering, access controls, encryption, and
    retention policies.
- Automate known, reversible responses.
  - Require preconditions, bounded scope, idempotency, timeout, rollback, and an
    audit trail.
  - Keep destructive or ambiguous remediation behind human approval until its
    safety has been demonstrated.
- Learn after operations.
  - Review incidents, near misses, capacity events, and alert noise.
  - Turn findings into owned improvements to code, alarms, runbooks, tests, and
    architecture.

## Operational readiness checklist

- [ ] Each critical service has objectives, dashboards, alerts, and an owner.
- [ ] Runbooks cover known events; playbooks cover investigative scenarios.
- [ ] Telemetry can connect a customer symptom to a change and dependency.
- [ ] Retention and access meet investigation and compliance needs.
- [ ] Automated actions are bounded, reversible, observable, and tested.
- [ ] Exercises verify paging, escalation, recovery, and evidence capture.

## References

- AWS Well-Architected Framework, [Operate](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/operate.html)
