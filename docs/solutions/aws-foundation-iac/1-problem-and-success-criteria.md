# Problem and Success Criteria

## Customer problem

An existing AWS environment contained business-serving network, Amazon EKS,
identity, DNS, certificate, and edge resources. Rebuilding them to introduce
Infrastructure as Code would have created unnecessary outage and replacement
risk. Continuing manual changes, however, made ownership, review, drift, and
recovery difficult to demonstrate.

The delivery goal was to adopt the effective estate into Terraform while
preserving resource identity and runtime behavior.

## Scope

In scope:

- VPC, public and private subnets, routes, egress, and EKS subnet tags;
- Amazon EKS cluster, managed node groups, and supported add-ons;
- cluster and workload IAM relationships;
- Route 53, ACM, CloudFront, and AWS WAF resources;
- remote state, pipeline checks, plans, approvals, and post-apply validation.

Out of scope:

- AWS Organizations and Control Tower account vending;
- organization-wide guardrails, Security Hub administration, or log archive;
- redesigning the network during import;
- claiming production-grade NAT availability for the non-production topology.

## Success criteria

| Criterion | Acceptance evidence |
|---|---|
| Ownership | Every in-scope resource maps to one Terraform root and state boundary |
| Safe adoption | Import and refresh do not propose unexplained recreation or replacement |
| Reviewability | Pipeline produces a readable plan before an authorized apply |
| Repeatability | Modules and roots declare pinned provider and dependency requirements |
| Runtime safety | AWS API state plus consuming workload checks pass after change |
| Recovery | State versioning, prior configuration, and resource-specific rollback are documented |
| Honest availability | Single points and accepted cost tradeoffs are named explicitly |

Percent improvements are intentionally absent because a comparable historical
delivery and incident baseline was not available.
