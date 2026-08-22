# Customer Delivery and Handoff

## Purpose

Treat consulting delivery as a transfer of an operable capability, not merely a
deployment. Success requires agreed outcomes, traceable decisions, verified
acceptance, trained owners, and a controlled transition to steady-state support.

## Practices

- Establish outcomes and evidence during discovery.
  - Capture business objectives, users, dependencies, constraints, risks,
    compliance needs, and current baselines.
  - Define acceptance tests and evidence owners before implementation begins.
- Mobilize a joint delivery model.
  - Agree decision rights, responsibilities, communication cadence, escalation,
    environments, access, and change controls.
  - Maintain a decision log, risk register, assumption register, and dependency
    map.
- Deliver in reversible increments.
  - Use pilots and migration waves with entry, exit, rollback, and stop criteria.
  - Demonstrate working increments to customer operators and incorporate their
    feedback.
- Make acceptance evidence-based.
  - Validate functional behavior, security, performance, resilience, recovery,
    cost visibility, and operational readiness against agreed criteria.
  - Record unmet criteria as owned follow-up work rather than silently treating
    them as accepted.
- Prepare the operating organization.
  - Provide architecture decisions, source ownership, inventories, dashboards,
    alerts, runbooks, playbooks, backup and restore procedures, and access paths.
  - Conduct role-based enablement and scenario exercises, including incident,
    rollback, and recovery drills.
- Close with explicit ownership.
  - Name service, platform, security, financial, and escalation owners.
  - Define support boundaries, warranty or hypercare period, success measures,
    and the process for future changes.

## Handoff acceptance record

| Area | Required evidence |
|---|---|
| Architecture | Current diagrams, decisions, dependencies, limits, and data flows |
| Delivery | Repositories, pipelines, environments, approvals, and rollback paths |
| Operations | Objectives, dashboards, alerts, runbooks, on-call, and escalation |
| Security | Identity, network, data, logging, findings, exceptions, and owners |
| Resilience | Backup scope, restore evidence, recovery objectives, and exercise record |
| Cost | Tags, budgets, usage drivers, optimization backlog, and accountable owner |
| Enablement | Training material, attendance, scenario results, and open questions |
| Acceptance | Signed criteria, exceptions, residual risks, and follow-up dates |

## References

- AWS Prescriptive Guidance, [The three-phase migration process](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-migration/overview.html)
- AWS Well-Architected Framework, [Operate](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/operate.html)
