# Enterprise AWS Networking and Security

## Purpose

Build network boundaries that support workload isolation, inspection,
connectivity, and recovery without relying on a flat VPC or manually maintained
rules. Network controls complement identity and application controls; they do
not replace them.

## Practices

- Start with account and environment boundaries.
  - Separate production, non-production, security, and log-archive concerns
    before choosing CIDR ranges or routing components.
  - Apply organization guardrails and centralize auditable security services.
- Design an IP addressing plan before deployment.
  - Reserve non-overlapping space for VPCs, subnets, Amazon EKS pods, hybrid
    connectivity, and future growth.
  - Track utilization so subnet or pod-address exhaustion is detected early.
- Use multiple Availability Zones and classify subnets by trust and route.
  - Keep internet-facing entry points separate from private workloads and data
    services.
  - Choose NAT gateways, VPC endpoints, and egress inspection based on
    availability requirements and cost, and document intentional compromises.
- Layer traffic controls.
  - Use route tables for reachability, security groups for workload boundaries,
    network ACLs only where stateless subnet controls are justified, and AWS WAF
    for supported application-layer threats.
  - Prefer private service access through VPC endpoints where it reduces public
    exposure and supports the required service behavior.
- Centralize connectivity and inspection when scale requires it.
  - Use AWS Transit Gateway or an equivalent governed topology instead of
    uncontrolled full-mesh peering.
  - Define who owns routing, DNS, firewall policy, certificates, and incident
    changes.
- Make network behavior observable.
  - Enable and retain appropriate VPC Flow Logs, load-balancer logs, AWS
    CloudTrail events, DNS query logs, and firewall findings.
  - Test accepted and rejected paths from the workload perspective.

## Architecture review questions

- Which accounts, VPCs, and subnets form trust boundaries?
- Where can traffic enter, leave, cross environments, or reach data services?
- Is every route and rule attributable to an owner and business requirement?
- What happens when one Availability Zone, NAT gateway, inspection path, or
  hybrid link fails?
- Can responders trace a denied or unexpected flow without enabling new logs?

## References

- AWS Well-Architected Framework, [Protecting networks](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/protecting-networks.html)
- AWS Management and Governance Guide, [Implementation priorities](https://docs.aws.amazon.com/wellarchitected/latest/management-and-governance-guide/implementation-priorities.html)
