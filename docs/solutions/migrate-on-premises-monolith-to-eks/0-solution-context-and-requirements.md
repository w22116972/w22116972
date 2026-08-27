# Solution Context and Requirements

## Abstract

This document defines the reference problem that the legacy-modernization
solution is designed to solve. It prevents the target architecture from being
interpreted as a universal EKS design and separates three kinds of statements:

- **Reference requirement** — an assumed enterprise need used to drive design;
- **Verified evidence** — supported by retained source, inventory, or career
  records; and
- **Acceptance target** — a value or gate that a real customer engagement must
  approve and measure before production.

The current solution covers an on-premises legacy application that does not
already have a production container and Kubernetes delivery contract. Migration
from an existing self-managed Kubernetes platform is a separate solution because
its source artifacts, compatibility risks, data movement, and rollback model are
different.

## Reference customer and program

| Area | Reference requirement | Status in this case study |
|---|---|---|
| Organization | Large enterprise with multiple application teams, shared platform/security/network functions, formal change control, and 24x7 production ownership | Design assumption, not a historical staffing claim |
| Portfolio | Enterprise portfolio contains at least 100 applications; this program modernizes one business-critical monolith into more than 20 deployable components | Portfolio size is a reference assumption; 20+ final components are retained evidence |
| Source platform | Application runs on on-premises virtual machines or bare metal and lacks a standard production container image, Helm chart, and Kubernetes operating contract | Reference source state; exact historical host inventory is not retained |
| Environments | Production and non-production require separate blast-radius and access boundaries | Reference requirement |
| AWS organization | Workloads operate inside an enterprise multi-account landing zone with centralized governance and workload accounts | Reference requirement; no completed landing-zone claim |
| Connectivity | Migration requires hybrid DNS, routing, firewall/inspection, identity, certificate, and dependency connectivity during coexistence | Reference requirement; real addresses and names are excluded |
| Availability | Production workloads span multiple Availability Zones; workload-specific SLO, RTO, and RPO values are approved by service tier | Reference requirement and acceptance target |
| Cluster topology | At least production and non-production cluster cells are separated; additional cells are selected by trust, regulation, lifecycle, quota, Region, or recovery independence | Reference requirement plus architecture decision |
| Delivery | Immutable images, versioned Helm releases, policy gates, traceable promotion, bounded rollout, and tested rollback are required | Retained mechanisms plus target controls |
| Security | Federated human access, workload identity, external secrets, least privilege, audit, supply-chain controls, and explicit network boundaries are required | Target controls; evidence must be produced per wave |
| Operations | Service and platform ownership, observability, incident response, upgrades, backup/recovery, capacity, cost, and handoff are production gates | Acceptance targets; historical sign-off is not retained |

“Large enterprise” is not treated as a reason to place everything in one large
cluster or to create one cluster per application. It means the assessment must
make account, cluster, tenant, quota, lifecycle, and failure-domain decisions
explicit and repeatable.

## Source-state definition

The mandatory path assumes:

- a legacy application with a coupled build, release, scale, and rollback unit;
- no approved immutable container image for each target deployable component;
- runtime configuration, filesystem, process, session, scheduler, or host
  assumptions that require discovery and remediation;
- a shared or tightly coupled data and integration model that must remain
  compatible during incremental extraction;
- incomplete per-capability ownership, health, telemetry, and resource data;
- delivery automation that does not yet provide source-to-running-image and
  chart traceability; and
- a retained source path that can remain available while bounded traffic waves
  are proven.

The source may include virtual machines, VMware, bare metal, appliances, or
managed dependencies. Rehosting infrastructure can be an interim bridge, but it
does not replace the containerization and application modernization path.

## Target-state requirements

### Business and migration outcomes

1. Move capabilities in dependency-aware waves rather than one irreversible
   application, data, platform, and traffic event.
2. Create useful independent release and scale boundaries where business and
   operational value justify the additional distributed-system cost.
3. Preserve customer behavior, data integrity, security controls, and an
   executable rollback path throughout coexistence.
4. Establish measurable service ownership and independent customer operation
   before delivery-team exit.
5. Retire legacy components only after traffic, consumers, data, recovery, and
   retention dependencies are cleared.

### Platform requirements

- AWS accounts, VPCs, subnets, connectivity, DNS, keys, certificates, logging,
  and identity are supplied by or integrated with the enterprise foundation.
- Production EKS cluster cells use multiple Availability Zones and private
  workload subnets with controlled ingress and egress.
- VPC CNI/IP-family, instance, `maxPods`, node-pool, storage, load-balancer, and
  quota decisions are derived from peak, release, maintenance, failure, and
  recovery scenarios.
- Cluster-scoped add-ons have one declarative owner, pinned compatibility, high
  availability where required, lifecycle policy, telemetry, and rollback.
- Production and non-production do not share a control plane unless an explicit
  risk decision overrides the reference requirement.
- Additional clusters are introduced only when an isolation, lifecycle, scale,
  Region, or recovery requirement exceeds the cost and operating impact.

### Application requirements

Each target deployable component must have:

- an immutable, minimally privileged image with documented base-image and
  vulnerability ownership;
- environment-neutral application code with external configuration and secret
  references;
- startup, readiness, liveness, graceful termination, and connection-draining
  behavior matched to real failure modes;
- measured CPU, memory, ephemeral storage, network, connection, and startup
  characteristics;
- an explicit Service, dependency, data, identity, storage, and telemetry
  contract;
- horizontally safe state, session, cache, lock, retry, scheduler, and message
  behavior, or a documented singleton/stateful design;
- a versioned Helm release with effective-manifest validation; and
- contract, integration, load, resilience, security, data, synthetic-journey,
  release, and rollback evidence appropriate to its service tier.

### Security and compliance requirements

- Human access uses federation, MFA, short-lived roles, cluster authorization,
  audit, and reviewed break-glass procedures.
- Workloads use dedicated Kubernetes ServiceAccounts and scoped temporary AWS
  credentials; node credentials are not the application permission model.
- Secrets remain outside images, source, Helm values, extracted diagnostics, and
  pipeline logs.
- Pod Security Standards, Linux security controls, RBAC, admission policy,
  default-deny network policy, egress controls, and supply-chain gates apply by
  service tier with governed exceptions.
- Data classification, encryption, residency, retention, deletion, backup, and
  evidence production are mapped to accountable owners.

### Reliability and operational requirements

- Every production service has numeric SLO, RTO, and RPO values approved in its
  private wave packet; this public reference does not invent them.
- Replica, Pod, node, IP, storage, connection, API, load-balancer, and
  downstream capacity is tested for normal peak, rollout surge, maintenance,
  selected failure, and recovery conditions.
- Traffic cutover uses a bounded route, tenant cohort, or weight with objective
  abort criteria, observation period, decision authority, and data-aware
  rollback.
- Dashboards begin with customer journeys and service objectives, then correlate
  application, Kubernetes, node, AWS, dependency, and delivery state.
- Platform and service teams own patching, Kubernetes and add-on lifecycle,
  certificates, secrets, capacity, quota, backup/restore, incidents, cost, and
  retirement.

## Scope boundaries

### Mandatory for this solution

- enterprise and application assessment;
- containerization and runtime remediation;
- EKS foundation and platform controls;
- Helm and CI/CD release contracts;
- capability extraction or bounded replatforming;
- data and dependency coexistence;
- traffic migration, validation, rollback, operations, and legacy retirement.

### Decision-gated modules

- VMware or server rehost bridge;
- database-engine modernization;
- serverless/event-driven extraction;
- service mesh;
- GitOps adoption;
- ingress-to-Gateway API migration;
- Karpenter, KEDA, or advanced elasticity;
- multi-Region disaster recovery;
- industry-specific controls; and
- agentic AIOps.

### Separate solution

Migration from an existing on-premises or self-managed Kubernetes cluster to
Amazon EKS is not modeled as “skip containerization” inside this lifecycle. It
requires its own source-cluster assessment, Kubernetes API and controller
compatibility, CNI/CSI and storage mapping, cluster-scoped resource ownership,
dual-cluster validation, and decommission sequence.

## Requirements traceability

| Requirement area | Primary lifecycle owner |
|---|---|
| Scenario, evidence, assumptions, scope | This document |
| Enterprise, portfolio, workload, capacity, and wave readiness | [Discovery and success criteria](1-discovery-and-success-criteria.md) and [enterprise assessment](1-1-enterprise-assessment.md) |
| Account, cluster, network, identity, security, capacity, and lifecycle design | [Target architecture and decisions](2-target-architecture-and-decisions.md) |
| Images, charts, pipelines, extraction, compatibility, and deployment | [Migration implementation](3-migration-implementation.md) |
| Load, resilience, traffic, data, rollback, and outcome evidence | [Cutover, validation, and results](4-cutover-validation-and-results.md) |
| Ownership, telemetry, runbooks, exercises, upgrades, and retirement | [Operations and handoff](5-operations-and-handoff.md) |

## Entry criteria

Use this solution when the source lacks a production Kubernetes workload
contract and material containerization or application-boundary work is required.
Use the separate self-managed-Kubernetes-to-EKS solution when the source already
runs containerized workloads under Kubernetes and the primary change is the
cluster platform, AWS integration, and cluster-to-cluster migration.

If the portfolio contains both source states, classify every application first
and run the two paths as coordinated workstreams. Do not force both through one
implementation checklist.
