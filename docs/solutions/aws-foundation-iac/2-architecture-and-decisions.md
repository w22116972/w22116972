# Architecture and Decisions

## Layered ownership

| Layer | Owner | Boundary |
|---|---|---|
| AWS organization | Customer governance | Account, organization policy, and central security extensions |
| AWS foundation | Terraform | Network, EKS, IAM, DNS, certificates, and edge controls |
| EKS managed services | Terraform through AWS APIs | Cluster, node groups, and managed add-ons |
| Kubernetes platform | GitOps controllers | Approved in-cluster declarations only |
| Secret values | External credential owner | Values are referenced, not copied into infrastructure source |

## State and module structure

Reusable modules contain focused capabilities. Environment roots select the
module versions and effective configuration for one lifecycle. Network, EKS,
cluster IAM, DNS, and edge resources do not share one monolithic state file.

This structure limits blast radius and credentials but requires outputs and
dependencies between roots to be explicit. Cross-root change order is part of
the runbook rather than hidden in a large dependency graph.

## Network decisions

- Two Availability Zones provide subnet and worker-placement diversity.
- Public and private routes distinguish ingress/edge components from workloads.
- Required EKS subnet tags are declared as part of the network contract.
- One NAT gateway is accepted for non-production cost control; production would
  require an availability and egress-cost decision based on recovery objectives.
- Default route table, security group, and network ACL adoption is avoided when
  Terraform cannot safely claim their complete shared lifecycle.

## Extension path

A broader enterprise landing zone can add AWS Organizations, Control Tower,
security and log-archive accounts, organization guardrails, centralized
inspection, and account vending. Those capabilities should be integrated
without reclassifying this environment implementation as proof that the wider
governance layer already exists.
