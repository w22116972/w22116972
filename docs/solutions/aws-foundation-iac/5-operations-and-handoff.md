# Operations and Handoff

## Operating model

- Platform owners approve module and root changes.
- Security owners review IAM, network exposure, encryption, and guardrail
  exceptions.
- Service owners validate workload behavior after infrastructure changes.
- The pipeline is the normal writer; emergency console changes require an
  incident record and prompt reconciliation.

## Routine controls

- Run scheduled read-only plans and assign every unexpected difference.
- Review provider, module, EKS, node, and add-on versions on a planned cadence.
- Monitor subnet/IP capacity, egress availability and cost, certificate expiry,
  DNS behavior, WAF findings, and state-access events.
- Exercise state recovery and at least one resource-specific rollback path.
- Reassess the single-NAT decision when availability or environment criticality
  changes.

## Handoff package

The customer receives the architecture and ownership maps, module and root
catalog, state inventory, pipeline and approval model, import records, exception
register, verification procedures, rollback runbooks, access model, and open
extension backlog.

Independent operation is accepted only after customer operators can plan a
change, interpret it, apply it through the authorized path, verify the effective
result, diagnose drift, and restore the prior state without the delivery team.
