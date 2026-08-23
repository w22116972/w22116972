# Legacy Application Migration and Modernization to Amazon EKS

## Executive summary

**Challenge:** A legacy monolith coupled application capabilities to one release,
scaling model, and operational blast radius. The target needed to support
independent delivery and scaling without combining platform, application, data,
and traffic risk in one big-bang rewrite.

**Approach:** I led an incremental replatform-and-refactor pattern: inventory
dependencies, establish an Amazon EKS foundation with Terraform, containerize
capabilities with multi-stage Docker builds, standardize application releases
with Helm and CI/CD, preserve data compatibility, and move bounded traffic waves
with explicit abort and rollback gates.

**Outcome:** The retained evidence supports a final estate of more than 20
containerized components on EKS and the Docker, Terraform, Helm, and pipeline
mechanisms used to deliver it. The current résumé also records 40% lower memory
consumption and 30% lower compute cost, but the source baseline, comparison
window, and allocation worksheet are not retained. Those percentages are
therefore disclosed as historical claims, not presented as verified results in
this case study.

## Outcome at a glance

| Area | Defensible result |
|---|---|
| Migration scale | Legacy monolith to a 20-plus-component containerized estate |
| Target platform | Amazon EKS with Terraform-owned foundation and Helm application releases |
| Delivery model | Independently versioned images and charts with build, scan, render, deploy, rollout, and traffic gates |
| Migration strategy | Replatform first, selectively refactor by dependency seam, retain data compatibility, cut over in bounded waves |
| Security | Workload identity, external secret references, image/dependency/manifest gates, and explicit network boundaries |
| Reliability | Health contracts, resource ownership, autoscaling review, real request-path validation, and tested rollback design |
| Operations | Service ownership, dashboards, runbooks, exercises, and independent-operation acceptance criteria |
| Resource outcome | 40% memory reduction is a historical résumé claim; calculation not retained, so not independently verified here |
| Financial outcome | 30% compute-cost reduction is a historical résumé claim; allocation/window not retained, so not independently verified here |
| Claims withheld | Downtime, deployment-frequency, latency, availability, rollback-time, and formal handoff improvements are not quantified |

## Architecture

```mermaid
flowchart LR
    U[Users and integrations] --> EDGE[DNS, TLS and load balancing]

    subgraph AWS[AWS account and VPC]
        EDGE --> GW[Ingress or Gateway]
        subgraph EKS[Amazon EKS across Availability Zones]
            GW --> A[service-a]
            GW --> B[service-b]
            GW --> C[service-c]
            A --> W[Workers and Jobs]
            B --> DATA[(Compatible data services)]
            C --> DATA
        end
        IAM[IAM and workload identity] --> EKS
        OBS[Metrics, logs, traces and alerts] --> EKS
        REG[Container registry] --> EKS
    end

    PR[Pull request] --> CI[Test, build, scan and render]
    CI --> REG
    CI --> HELM[Versioned Helm release]
    HELM --> EKS
    TF[Terraform] --> AWS
    EKS --> V[Rollout, dependency, traffic and data verification]
```

The architecture uses symbolic names and intentionally excludes internal
accounts, repositories, endpoints, IPs, and customer identifiers.

## Why EKS

- **Independent delivery and scaling:** The 20-plus-component target needed
  separate release, resource, health, and scaling boundaries instead of the
  monolith's shared lifecycle.
- **One Kubernetes delivery model:** Existing Kubernetes skills, Helm packages,
  and policy controls could serve mixed-runtime services without introducing a
  second orchestration model.
- **AWS integration with less control-plane ownership:** EKS provides a managed
  Kubernetes control plane while integrating with IAM, VPC networking, load
  balancing, and AWS observability services.
- **Extensible platform standards:** Kubernetes APIs enabled reusable security,
  deployment, autoscaling, and operational patterns across teams and services.

This choice was made for the aggregate platform, not because every individual
service requires Kubernetes. It accepted more operational responsibility than
ECS, including add-on upgrades, worker capacity, scheduling, networking,
observability, backup, and operator enablement.

## Migration flow

1. **Discover and baseline.** Inventory dependencies, traffic, data ownership,
   service objectives, current resource use, and rollback constraints.
2. **Establish the platform.** Provision the EKS, IAM, networking, registry,
   observability, and delivery foundations before moving application traffic.
3. **Define the deployable contract.** Give each selected component a versioned
   image, Helm chart, configuration and secret interface, health checks,
   resource policy, ownership, and rollback procedure.
4. **Migrate one bounded capability.** Select a dependency seam, preserve
   compatible data paths, and deploy the component without immediately removing
   its legacy path.
5. **Prove production behavior.** Run contract, integration, load, failure,
   security, data-reconciliation, and end-to-end request-path tests.
6. **Shift traffic in a bounded wave.** Move a route, tenant cohort, or traffic
   percentage; observe agreed service and dependency thresholds; then advance
   or roll back.
7. **Stabilize, hand off, and retire.** Complete an observation window, prove
   independent operation and rollback, and remove the legacy path only after
   its exit criteria are met.

Steps 3–6 repeat for each migration wave. The promotion gates are intentionally
separate: a built image is not a deployed release; a deployed release is not a
healthy workload; and a Ready workload does not prove that real traffic,
dependencies, identity, and data behave correctly.

## Key decisions and tradeoffs

| Decision | Reason | Tradeoff |
|---|---|---|
| Replatform before deep refactoring | Separated platform risk from application and data redesign | Temporary legacy models and adapters remained |
| Extract by capability and dependency seam | Created useful release and scaling boundaries | Required contract discovery and coexistence logic |
| One image and chart per deployable component | Made identity, ownership, rollback, and resources explicit | Increased repository/release count and need for shared standards |
| Retain compatible data paths during waves | Kept rollback available without immediate reverse migration | Extended expand-and-contract periods |
| Terraform for AWS/EKS, Helm for applications | Matched lifecycle and blast radius | Cross-layer changes required coordinated sequencing |
| Traffic and data proof above pod readiness | Caught integration failures that Kubernetes health alone cannot see | Added synthetic, contract, and reconciliation work |

## Implementation problem

A service could complete its Kubernetes rollout while still using a wrong
downstream endpoint inherited from a legacy/default configuration. The pods were
running, but the user journey failed when the dependency was called.

The correction was to externalize and explicitly set environment endpoints,
test the dependency contract, trace callers and dependencies together, and add a
real synthetic journey to the post-deploy gate. Adding the dependency to
liveness was rejected because restarting healthy processes would amplify a
configuration or downstream outage.

## Validation and rollback

Each wave combines:

- unit, contract, integration, functional-parity, data, load, resilience,
  deployment, and security evidence;
- pinned source, image digest, chart version, rendered manifest, and effective
  runtime identity;
- service-objective and dependency thresholds approved before cutover;
- a bounded route, tenant cohort, or traffic weight;
- route, release, and data-aware rollback; and
- an observation window covering representative business demand.

Expansion stops on service-objective breach, data-integrity failure, dependency
saturation, unhealthy scaling or scheduling, missing telemetry, release identity
mismatch, or loss of the tested rollback path.

## Results and measurement limits

The public evidence supports the platform change and final service scale. It
does not support independently replaying every historical percentage:

| Metric | Before | After | Change | Window | Evidence |
|---|---:|---:|---:|---|---|
| Application shape | Legacy monolith | 20+ containerized components on EKS | Independent packaging and release boundaries | Historical program; final snapshot retained | Résumé and sanitized source inventory |
| Delivery mechanism | Coupled legacy delivery | Docker, Terraform, Helm, and CI/CD patterns | Standardized release contract | Final retained snapshot | Source artifacts |
| Memory consumption | Not retained | Not retained | 40% lower recorded in résumé | Not retained | Historical claim only |
| Compute cost | Not retained | Not retained | 30% lower recorded in résumé | Not retained | Historical claim only |

A future verified percentage requires matched periods and a retained calculation:

```text
resource intensity = aggregate resource consumption / successful business output
unit cost          = eligible compute cost / successful business output
improvement        = (baseline - comparison) / baseline
```

The worksheet must preserve scope, dates, demand, percentiles, cost allocation,
discount treatment, confounders, source queries, and reviewers.

## Customer enablement

The target operating model assigns every service a product owner, technical
owner, on-call path, objective, dashboard, dependency map, release and rollback
runbook, data/recovery dependency, and debt exit criteria. Handoff progresses
from walkthrough to guided operation, paired execution, failure exercise,
independent deployment/rollback, and independent incident response.

The retained record does not contain a historical handoff scorecard, so this
case study defines those acceptance gates without claiming a measured training
or independence result.

## Modernization roadmap

```mermaid
flowchart LR
    P1["1 · Discover<br/>portfolio and migration evidence"]
    P2["2 · Establish<br/>platform and governance"]
    P3["3 · Migrate and modernize<br/>application and data"]
    P4["4 · Validate and cut over<br/>traffic, resilience and results"]
    P5["5 · Operate and improve<br/>elasticity, lifecycle and adoption"]
    P1 --> P2 --> P3 --> P4 --> P5

    D["Discovery and tracking tooling"] -.-> P1
    S["Identity · supply chain · tenancy · VMware bridge"] -.-> P2
    A["GitOps · database · serverless · service mesh · industry patterns"] -.-> P3
    V["DR · Envoy Gateway · sustainability"] -.-> P4
    O["Karpenter · autoscaling · FinOps · SRE · upgrades · platform engineering · AIOps"] -.-> P5
```

These extensions are decision-gated. They are not a requirement to adopt every
technology, and a design or assessment is not presented as an implemented
production result. The parent phases already contain the core migration story;
the extension numbers preserve lower subphase slots for the separately designed
core topics.

## Detailed implementation

### Parent phases

1. [Discovery and success criteria](1-discovery-and-success-criteria.md)
2. [Target architecture and decisions](2-target-architecture-and-decisions.md)
3. [Migration implementation](3-migration-implementation.md)
4. [Cutover, validation, and results](4-cutover-validation-and-results.md)
5. [Operations and handoff](5-operations-and-handoff.md)

### Phase 1 extensions: discovery

- [Migration discovery and tracking tooling](1-5-migration-discovery-and-tracking-tooling.md)

### Phase 2 extensions: platform and governance

- [Workload identity and secrets modernization](2-5-workload-identity-and-secrets-modernization.md)
- [Software supply chain and runtime security](2-6-software-supply-chain-and-runtime-security.md)
- [Amazon EKS multi-tenancy and governance](2-7-eks-multi-tenancy-and-governance.md)
- [VMware rehost or relocate bridge to Amazon EKS](2-8-vmware-rehost-relocate-bridge-to-eks.md)

### Phase 3 extensions: application and data

- [Automated testing, CI/CD, and GitOps modernization](3-4-automated-testing-cicd-and-gitops.md)
- [Database modernization](3-5-database-modernization.md)
- [Serverless and event-driven extraction](3-6-serverless-and-event-driven-extraction.md)
- [Service mesh and east-west traffic modernization](3-7-service-mesh-and-east-west-traffic.md)
- [Industry-specific workload architecture](3-8-industry-specific-workload-architecture.md)

### Phase 4 extensions: validation and cutover

- [Reliability and disaster recovery modernization](4-3-reliability-and-disaster-recovery-modernization.md)
- [NGINX Ingress to Envoy Gateway modernization](4-4-nginx-ingress-to-envoy-gateway-modernization.md)
- [Sustainability and resource-efficiency review](4-5-sustainability-and-resource-efficiency-review.md)

### Phase 5 extensions: operations and continuous improvement

- [Karpenter compute and elasticity modernization](5-2-karpenter-compute-and-elasticity-modernization.md)
- [Workload autoscaling modernization](5-3-workload-autoscaling-modernization.md)
- [Amazon EKS FinOps and cost optimization](5-4-eks-finops-and-cost-optimization.md)
- [Observability and SRE operating model](5-5-observability-and-sre-operating-model.md)
- [Amazon EKS lifecycle and upgrade engineering](5-6-eks-lifecycle-and-upgrade-engineering.md)
- [Platform engineering and golden paths](5-7-platform-engineering-and-golden-paths.md)
- [Agentic AIOps with Amazon Bedrock](5-8-agentic-aiops-with-amazon-bedrock.md)
- [Customer expansion and modernization roadmap](5-9-customer-expansion-and-modernization-roadmap.md)

Related case-study deep dives: [GitOps platform modernization](../gitops-platform-modernization/README.md),
[EKS reliability and cost optimization](../eks-reliability-cost-optimization/README.md),
[EKS disaster recovery](../eks-disaster-recovery/README.md),
[database migration and modernization](../database-migration-modernization/README.md),
[serverless application modernization](../serverless-application-modernization/README.md),
and [Agentic AIOps](../agentic-aiops-bedrock/README.md).
