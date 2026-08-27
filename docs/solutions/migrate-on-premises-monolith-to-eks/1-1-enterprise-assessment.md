# Enterprise Migration and Amazon EKS Readiness Assessment

## Abstract

This assessment is designed for a large enterprise portfolio, where application
count, organizational boundaries, regulatory obligations, shared networks, and
failure blast radius matter as much as container compatibility. It converts an
initial inventory into decisions and measurable entry criteria for the Amazon
EKS foundation and each migration wave.

## Evidence standard

The assessment model is a recommended delivery contract. It does not imply that
every worksheet or measurement was retained from the historical migration. Any
public result must still be labeled as verified evidence, a historical claim, or
an acceptance target.

## Assessment model

Assessment is continuous rather than a one-time questionnaire:

1. **Portfolio assessment** establishes breadth, business priorities, broad
   dependencies, migration strategies, and a directional business case.
2. **Platform assessment** establishes enterprise account, cluster, network,
   identity, security, reliability, quota, and operating-model constraints.
3. **Application assessment** develops implementation-level detail for the next
   migration waves.
4. **Wave reassessment** updates assumptions using measurements and incidents
   from completed waves before later waves are approved.

## Required assessment outcomes

| Outcome | Required decision or artifact | Accountable role |
|---|---|---|
| Business alignment | Approved outcomes, service objectives, constraints, funding model, and decision authorities | Executive sponsor and product leadership |
| Portfolio baseline | Reconciled application, infrastructure, data, owner, lifecycle, criticality, and environment inventory | Program and application portfolio owners |
| Migration strategy | A documented retain, retire, rehost, relocate, repurchase, replatform, or refactor decision for every in-scope application | Application and enterprise architects |
| Dependency model | Runtime, data, identity, network, certificate, DNS, batch, vendor, and operational dependencies with confidence labels | Application and data owners |
| Enterprise topology | Account, Region, cluster, environment, tenant, namespace, and failure-domain boundaries | Enterprise and platform architects |
| Capacity model | Peak and failure-state Pod, node, IP, storage, load-balancer, API, and service-quota demand | Platform and network owners |
| Security and compliance | Data classification, control mapping, identity model, evidence owner, exception process, and residual-risk approval | Security, risk, and compliance owners |
| Reliability model | SLOs, RTO/RPO, failure scenarios, recovery dependencies, rollback limits, and test plan | Service and resilience owners |
| Delivery readiness | Build, artifact, chart, policy, environment promotion, release, and rollback contracts | Platform and service delivery owners |
| Operating readiness | Ownership, telemetry, incident response, support coverage, upgrade, patch, and lifecycle obligations | Operations and service owners |
| Business case | Baseline run cost, migration cost, target run cost, allocation rules, benefits, risks, and sensitivity ranges | Finance partner and program owner |
| Wave plan | Dependency-aware groups with entry criteria, exit criteria, rollback boundaries, and named approvers | Migration lead |

No application enters a production wave merely because it has a Dockerfile or
runs successfully in a development cluster.

## Enterprise discovery domains

| Domain | Questions that must be answered | Evidence to retain |
|---|---|---|
| Business | Which journeys and deadlines matter? What revenue, safety, contractual, or regulatory impact follows an outage? | Business capability map, critical periods, approved objectives, decision log |
| Portfolio | What exists, who owns it, which version is supported, and when does it retire? Are CMDB, billing, source, DNS, and runtime inventories consistent? | Reconciled inventory with source and confidence per field |
| Application | Is the process stateless? Can it run multiple replicas? How does it start, stop, drain, retry, and recover? | Runtime profile, code/config review, representative tests |
| Dependencies | Which synchronous, asynchronous, data, identity, DNS, certificate, vendor, and operator dependencies exist? | Observed flows, traces, connection data, owner-validated map |
| Data | Who owns the data? What are its classification, locality, consistency, retention, encryption, backup, and schema constraints? | Data inventory, flow map, RTO/RPO, restore and reconciliation plan |
| Runtime capacity | What are peak demand, concurrency, latency, CPU, memory, ephemeral storage, network, GPU, and startup characteristics? | Matched-period telemetry, load profiles, tested per-Pod capacity |
| Network | Which CIDRs overlap? Where are DNS, proxies, firewalls, NAT, endpoints, private links, ingress, egress, and inspection boundaries? | IPAM inventory, flow logs, route and firewall review, bandwidth and latency tests |
| Identity | How do humans, workloads, pipelines, and external systems authenticate and receive authorization? | Role and permission inventory, trust paths, denied/allowed tests |
| Security | Which threat boundaries, vulnerabilities, admission controls, runtime controls, and supply-chain requirements apply? | Threat model, control matrix, exception register, security test plan |
| Compliance | Which controls require segregation, immutability, retention, residency, approval, or evidence production? | Regulation-to-control mapping and evidence ownership |
| Reliability | Which failures must be tolerated: process, node, zone, dependency, network, identity, control plane, data, or Region? | SLOs, error budgets, failure-mode analysis, exercise schedule |
| Delivery | Can source be built reproducibly and promoted without rebuilding? Are database and application changes backward compatible? | Pipeline map, artifact lineage, release and rollback evidence |
| Operations | Who owns incidents, capacity, certificates, secrets, upgrades, add-ons, nodes, policies, backup, and cost? | RACI, support model, runbooks, escalation and maintenance calendar |
| Organization | Are platform, application, security, network, data, finance, and support teams staffed and empowered to make decisions? | Skills assessment, enablement backlog, named owners and deadlines |
| Commercial | Do licenses, vendor support, appliances, data transfer, support plans, or contractual terms constrain the target? | Contract and license review, cost model, vendor decision log |

Unknown values remain visible as risks. They are not silently replaced with
generic defaults.

## Workload assessment record

Each deployable workload needs a record that is detailed enough to size,
schedule, secure, test, operate, and roll back it.

| Field group | Required data |
|---|---|
| Identity | Symbolic workload ID, business capability, owner, repository, runtime, version, lifecycle, environments |
| Criticality | Business impact, tier, SLO, critical periods, RTO, RPO, recovery ordering |
| Demand | Baseline, normal peak, exceptional peak, seasonality, request rate, concurrency, queue depth, batch schedule |
| Replica behavior | Minimum and maximum replicas, tested capacity per replica, startup time, termination grace, disruption tolerance |
| Resources | CPU and memory requests/observed percentiles, limits rationale, ephemeral storage, GPU, huge pages, architecture constraints |
| Network | Listeners, protocols, connection count, bandwidth, DNS, ingress, egress, dependencies, fixed-IP assumptions, source-IP requirements |
| State and storage | Persistent paths, volume mode, throughput, IOPS, latency, access mode, topology, backup and restore behavior |
| Configuration | Environment-specific values, feature flags, secret references, certificate ownership, reload and rotation behavior |
| Security | ServiceAccount, AWS permissions, Kubernetes permissions, Pod security context, image provenance, data classification |
| Delivery | Build duration, artifact identity, chart, schema sequence, release frequency, maintenance window, rollback mechanism |
| Observability | Service and dependency indicators, logs, metrics, traces, audit events, alert and dashboard ownership |
| Migration | Selected strategy, dependency group, coexistence design, data sequence, traffic unit, abort threshold, legacy retirement gate |

Twenty application components do not mean twenty Pods. Deployments can have
many replicas, Jobs can overlap, DaemonSets grow with node count, and platform
controllers also consume cluster capacity.

## Workload and Pod demand model

### Establish measured per-replica capacity

For each horizontally scalable workload, determine the sustainable capacity of
one replica under a representative payload. The test must include dependency
latency, sidecars, encryption, logging, retries, and connection pools rather
than measuring an isolated handler.

```text
load-based replicas = ceiling(peak demand / tested sustainable demand per Pod)
required replicas   = max(load-based replicas, availability minimum)
```

The sustainable value is below the point where the workload breaches its
latency, error, saturation, queue, or dependency threshold. HPA `maxReplicas`
must cover the approved peak and failure scenarios; it is not evidence that the
application can safely scale that far.

### Model concurrent scenarios

Capacity is assessed for each scenario, not only for an average day:

| Scenario | Include |
|---|---|
| Steady state | Minimum and normal replicas, platform services, DaemonSets, routine Jobs |
| Business peak | Peak request/concurrency profile, expected HPA maxima, queue consumers, known batch overlap |
| Release | Old and new ReplicaSets, `maxSurge`, canary replicas, migration Jobs, test traffic |
| Node maintenance | Drained node capacity, disruption budgets, rescheduling time, replacement startup |
| Availability Zone loss | Pods and supporting capacity that must run in the surviving zones, plus traffic redistribution |
| Dependency degradation | Increased latency, retries, circuit breakers, queue growth, and connection retention |
| Recovery | Restore/reconciliation Jobs, replayed messages, caches warming, elevated support telemetry |
| Exceptional event | Approved demand spike, node scale-out rate, load-balancer warming, API and downstream limits |

For each Availability Zone and node pool:

```text
peak workload Pods
  = sum(maximum concurrent replicas by workload)
  + concurrent Jobs and workers
  + release surge and canary Pods

total scheduled Pods
  = peak workload Pods
  + platform Deployments
  + DaemonSets per node
  + agreed spare capacity

nodes required for a scenario = maximum of:
  ceiling(total Pod CPU requests / allocatable CPU per node)
  ceiling(total Pod memory requests / allocatable memory per node)
  ceiling(total Pod ephemeral-storage requests / allocatable storage per node)
  ceiling(Pods requiring this node pool / effective Pods per node)
  minimum nodes required by zone, topology and disruption policy
```

Calculate this per compatible node pool: a total cluster-wide surplus cannot
schedule a Pod whose architecture, accelerator, storage topology, taint,
affinity, security, or zone constraints only match an exhausted pool. DaemonSet
resource demand is recalculated as node count changes because each added node
also adds its node-local agents.

Headroom is an explicit, reviewed quantity tied to node launch time, Pod startup
time, traffic growth, failure recovery, and business risk. It is not an
unexplained percentage copied between clusters.

## Amazon VPC CNI and IP capacity assessment

### Why Pod demand and IP demand are separate

With the default Amazon VPC CNI, ordinary Pods receive addresses from VPC
subnets. A node can therefore have free CPU and memory but still be unable to
schedule another Pod because its ENI/IP capacity or the subnet address pool is
exhausted. Conversely, the CNI warm pool can reserve more subnet addresses than
the number of currently running Pods.

Track at least these related but different values:

```text
workload demand       = replicas needed to meet service objectives
scheduler demand      = workload Pods + platform Pods + DaemonSet Pods
Pod IP demand         = Pods that require VPC addresses + CNI warm capacity
subnet address demand = Pod IPs + node ENIs + load balancers + endpoints
                        + EKS-managed interfaces + other VPC consumers
```

### Node-level Pod capacity

The effective Pod capacity of a node is the lowest applicable constraint:

```text
effective Pods per node = minimum of:
  kubelet maxPods
  VPC CNI address capacity for the instance and CNI mode
  CPU, memory and ephemeral-storage scheduling capacity
  ENI, branch-ENI, volume-attachment and device constraints
  organization policy or tested operating limit
```

The assessment records:

- every allowed instance type and its ENI/IP limits;
- secondary-IP, prefix-delegation, custom-networking, or IPv6 mode;
- kubelet `maxPods` and how it is configured;
- whether Security Groups for Pods consume branch ENI capacity;
- CNI `WARM_IP_TARGET`, `MINIMUM_IP_TARGET`, `WARM_PREFIX_TARGET`, or
  `WARM_ENI_TARGET` behavior;
- mixed-instance node-group behavior and the lowest advertised Pod capacity;
- volume-attachment, conntrack, packets-per-second, and bandwidth constraints;
  and
- the maximum Pods allowed by resource requests before IP capacity is reached.

Prefix delegation can increase IPv4 Pod density by allocating `/28` prefixes
to ENIs, but the subnet must have sufficiently large contiguous blocks. Free
but fragmented addresses do not guarantee that a prefix can be allocated.
Subnet CIDR reservations or new workload subnets are considered before enabling
prefix mode on a heavily fragmented network.

### Subnet and Availability Zone worksheet

IP capacity is calculated independently for every subnet and Availability
Zone. Spare addresses in one zone cannot satisfy a failed allocation in
another.

| Consumer | Steady state | Peak or failure state | Source of quantity |
|---|---:|---:|---|
| Node primary and additional ENIs | TBD | TBD | Maximum nodes by pool and instance ENI behavior |
| Pod addresses or delegated prefixes | TBD | TBD | Scenario Pod demand and CNI warm-pool settings |
| EKS control-plane cross-account ENIs | TBD | TBD | Cluster subnet and upgrade design |
| Load balancers and targets | TBD | TBD | Ingress/Gateway and service exposure inventory |
| Interface VPC endpoints | TBD | TBD | Private-service access design |
| NAT, inspection, proxies, and appliances | TBD | TBD | Enterprise egress and security architecture |
| Other shared VPC consumers | TBD | TBD | IPAM and infrastructure inventory |
| Reserved growth and replacement capacity | TBD | TBD | Growth forecast and simultaneous replacement plan |
| Available addresses after AWS reservation | TBD | TBD | Deployed subnet or approved target CIDR |

The worksheet must prove all of the following:

- normal peak fits without consuming the operational reserve;
- rolling node and cluster upgrades can allocate replacement interfaces before
  old capacity is removed;
- release surge and node autoscaling can occur concurrently;
- the surviving zones can host the required service tier after the selected
  zone-failure scenario;
- CNI warm allocation cannot unexpectedly consume the remaining subnet space;
- address overlap does not block connectivity to retained environments,
  partners, acquisitions, or on-premises networks; and
- dashboards and alerts exist for available subnet addresses, CNI allocation
  failures, ENI errors, and unschedulable Pods.

IPv4 secondary CIDRs, custom networking, prefix delegation, and IPv6 are design
options, not automatic fixes. The decision must consider routing, enterprise
IPAM, inspection, partner connectivity, application compatibility, operating
complexity, and the fact that a cluster's IP family is a foundational choice.

## Cluster scale, Kubernetes API, and churn assessment

Enterprise scale is multi-dimensional. Node and Pod counts alone do not
describe pressure on the Kubernetes control plane. The assessment records:

- projected nodes, Pods, namespaces, Services, EndpointSlices, Secrets,
  ConfigMaps, custom resources, and events;
- steady-state and burst creation/deletion rates for Pods, nodes, Jobs, and
  endpoints;
- controller, operator, policy engine, GitOps, autoscaler, and monitoring API
  clients;
- list versus watch behavior, informer use, retry/backoff, client timeouts, and
  API Priority and Fairness expectations;
- mutating and validating webhook count, timeout, availability, match scope,
  and failure policy;
- short-lived Job volume and retention;
- DNS query volume, service discovery behavior, and EndpointSlice use;
- node and Pod scale-out rate, image-pull throughput, registry concurrency, and
  load-balancer readiness; and
- etcd/object growth indicators and control-plane metrics that require alerts.

High churn can be more dangerous than a larger but stable cluster. Burst tests
must exercise the complete chain: demand signal, workload autoscaler, node
provisioner, EC2 APIs, VPC CNI, DNS, image registry, storage, load balancer,
admission, scheduler, application startup, and downstream capacity.

For a projected large cluster, record whether one cluster reduces operational
duplication enough to justify its larger blast radius. Separate clusters are
considered when required by hard tenant isolation, account or quota boundaries,
Region, lifecycle cadence, regulatory scope, incompatible cluster-wide
components, or recovery independence. A namespace is not treated as a hard
security boundary.

## AWS and Kubernetes quota register

Quotas are captured from the target account and Region; documentation defaults
are not used as proof of available capacity. Each entry records current usage,
current quota, forecast peak, requested quota, adjustability, request lead time,
owner, approval state, alarm, and fallback.

At minimum, assess applicable quotas and throttles for:

- EKS clusters, managed node groups, Fargate profiles, access entries, and Pod
  Identity associations;
- EC2 On-Demand and Spot capacity, vCPUs, ENIs, private addresses, API request
  rates, launch templates, and Auto Scaling groups;
- VPCs, subnets, route tables, routes, security groups, rules, prefix lists,
  NAT gateways, Transit Gateway attachments, and VPC endpoints;
- Elastic Load Balancing load balancers, target groups, targets, rules,
  certificates, and API rates;
- IAM roles, policies, policy attachments, OIDC providers, and STS request
  rates;
- EBS volumes, snapshots, throughput, IOPS, attachment limits, and CSI API
  behavior;
- Route 53 hosted zones, records, health checks, Resolver endpoints, and API
  request rates;
- ECR repositories, pull-through behavior, storage, pull rates, and cross-Region
  requirements;
- CloudWatch metrics, logs, alarms, dashboards, ingestion, retention, and API
  rates;
- KMS, Secrets Manager, Certificate Manager, S3, backup services, queues,
  databases, and any workload-specific dependency; and
- Kubernetes object, control-plane storage, DNS, conntrack, and node operating
  limits that are not represented in AWS Service Quotas.

Quota increases and capacity reservations must be approved before the wave that
depends on them. A submitted request is not available capacity.

## Enterprise account, cluster, and tenancy assessment

The topology decision is made from isolation and lifecycle requirements, not
from a default preference for one large cluster or one cluster per application.

| Boundary question | Cluster/account separation is favored when | Shared cluster can be considered when |
|---|---|---|
| Trust | Tenants are untrusted or require a strong security boundary | Tenants are similarly trusted and compensating controls are accepted |
| Environment | Production must be isolated from lower environments | Risk and compliance explicitly allow sharing |
| Regulation/data | Controls, residency, evidence, keys, or administrators must differ | Control scope and ownership are genuinely common |
| Blast radius | Failure or operator error must not affect another business service | Shared failure domain fits approved impact tolerances |
| Lifecycle | Kubernetes/add-on versions or maintenance windows conflict | Consumers can adopt a shared lifecycle |
| Scale/quota | A single account or cluster would approach tested limits | Aggregate demand stays within tested limits and reserves |
| Network | Routes, CIDRs, inspection, or connectivity models conflict | Network policy and routing requirements are compatible |
| Cost/operations | Duplication is justified by isolation or autonomy | Shared platform materially reduces safe operating effort |

The selected design defines:

- AWS Organizations and account placement;
- production and non-production separation;
- Region and disaster-recovery relationships;
- cluster purpose and ownership;
- tenant-to-account, cluster, and namespace mapping;
- cluster administrator and break-glass boundaries;
- resource quotas, limit ranges, priority classes, and fair-use controls;
- chargeback/showback labels and allocation boundaries; and
- a documented threshold for splitting or adding clusters.

## Security, risk, and compliance assessment

Security assessment must cover the complete delivery and runtime path:

| Area | Assessment requirement |
|---|---|
| Human access | Federation, MFA, role separation, cluster authorization, privileged approval, session/audit evidence, break-glass expiry and review |
| Workload identity | Service-specific ServiceAccounts, EKS Pod Identity or IRSA trust, cross-account access, session scope, denied and allowed actions |
| Kubernetes authorization | Namespace and cluster RBAC, impersonation risk, service-account token use, aggregated roles, escalation paths |
| Pod/runtime security | Pod Security Standards, Linux identity, capabilities, seccomp, read-only filesystem, host namespace/path/device use, privileged exceptions |
| Network security | Ingress, east-west, egress, DNS, metadata access, security groups, NetworkPolicies, inspection, flow evidence |
| Secrets and keys | External authority, encryption, rotation, reload, revocation, access audit, backup treatment, log and crash-dump exposure |
| Supply chain | Source controls, dependency and secret scanning, SBOM, image provenance/signing, registry policy, manifest and IaC policy, exception expiry |
| Admission | Policy ownership, fail-open/fail-closed choice, availability, performance budget, emergency bypass and audit |
| Node and add-ons | Supported OS and architecture, immutable or controlled access, patch SLA, add-on ownership, compatibility and vulnerability response |
| Audit and evidence | EKS control-plane logs, AWS API audit, workload events, retention, integrity, access, cost and evidence-production test |
| Data protection | Classification, encryption, residency, retention, deletion, backup, restore, tokenization and non-production data controls |

Every exception has a named risk owner, compensating control, expiry date, and
removal plan. “Runs in a private subnet” is not accepted as a complete security
control.

## Application and data modernization readiness

### Application checks

- deterministic, non-interactive build from an approved and supported base;
- configuration separated from the image and secrets referenced externally;
- no hidden dependency on a writable container filesystem or fixed host;
- graceful termination, connection draining, bounded retries, idempotency, and
  duplicate-message handling;
- startup, readiness, and liveness semantics tied to distinct failure modes;
- horizontal-scale safety for sessions, caches, locks, schedulers, and singleton
  work;
- explicit CPU, memory, ephemeral-storage, network, and storage behavior;
- supported CPU architecture, kernel, library, device, and licensing needs;
- application and sidecar telemetry that does not overload the service or
  control plane; and
- a tested image, configuration, schema, traffic, and data rollback boundary.

### Data and dependency checks

- system of record, data owner, readers, writers, schemas, consistency, and
  transaction boundaries;
- data volume, growth, change rate, transfer window, bandwidth, encryption, and
  reconciliation method;
- forward and backward schema compatibility while legacy and EKS paths coexist;
- RTO/RPO, backup immutability, restore ordering, restore capacity, and
  application-level read/write validation;
- connection limits, pools, timeouts, retries, circuit breakers, DNS caching,
  certificates, and dependency maintenance windows;
- cross-zone, cross-Region, internet, hybrid, inspection, and NAT data-transfer
  paths and cost; and
- external vendor capacity, allowlists, rate limits, support and rollback.

A completed data-transfer or restore job proves execution of a mechanism; it
does not prove consistency or usable application recovery.

## Reliability and cutover assessment

Each service tier defines:

- service-level indicators, objectives, error budget, measurement window, and
  decision owner;
- minimum replicas and zone distribution for normal and failure states;
- node, zone, network, DNS, identity, dependency, data, and control-plane
  failure behavior;
- Pod disruption, maintenance, upgrade, Spot interruption, and node-replacement
  policy;
- RTO/RPO and the exact recovery sequence for application, data, identity,
  network, configuration, and traffic;
- cutover unit such as route, tenant, cohort, or traffic percentage;
- abort thresholds, observation duration, rollback authority, and last safe
  rollback point; and
- load, resilience, restore, rollback, and game-day evidence required before
  production.

Readiness, replica count, topology spread, and a PodDisruptionBudget are useful
controls but do not independently prove availability. The evidence must include
real request paths and the failure states the architecture claims to tolerate.

## Delivery, platform, and operational readiness

### Delivery readiness

- protected source and review paths with traceable approvals;
- reproducible build, tests, dependency checks, secret checks, SBOM, image scan,
  signature/provenance, and immutable artifact identity;
- Helm render, schema, policy, and Kubernetes-version compatibility checks;
- separate infrastructure, add-on, application, schema, and traffic lifecycles;
- environment promotion of the same artifact rather than rebuilds;
- concurrency control, deployment identity, rollback, and post-deploy traffic
  verification; and
- emergency-change procedure with bounded authority and retrospective review.

### Platform and operational readiness

- landing-zone, account, IPAM, DNS, certificate, key, registry, identity,
  logging, monitoring, backup, and connectivity prerequisites;
- platform API/service catalog and golden paths with an exception mechanism;
- ownership and lifecycle for Kubernetes, node images, CNI, CSI, DNS, ingress or
  Gateway, autoscaling, policy, observability, backup, and GitOps components;
- metrics, logs, traces, events, audit, dashboards, alerts, synthetic tests, and
  cost visibility with retention and access controls;
- incident, escalation, vendor-support, maintenance, vulnerability, certificate,
  secret, capacity, quota, upgrade, and disaster procedures;
- production access and troubleshooting paths that do not depend on one person;
  and
- capacity and cost review, patch and upgrade calendar, recovery exercises, and
  service-owner acceptance.

## Financial and commercial assessment

The business case separates one-time transformation cost from ongoing platform
and workload cost. It includes ranges and sensitivity rather than one optimistic
point estimate.

| Cost or value area | Include |
|---|---|
| Baseline | Current compute, licenses, support, facilities/cloud, network, storage, backup, operations, incidents, and delivery effort |
| Migration | Discovery, remediation, dual running, data transfer, tooling, training, testing, support, contingency, and retirement |
| Target platform | EKS, nodes, load balancing, NAT/inspection, storage, logs/metrics/traces, backup, security services, support, and shared platform labor |
| Workload | Requests and actual use, replicas, environments, data transfer, storage, managed dependencies, licenses |
| Benefits | Faster independent change, resilience, elasticity, retirement, risk reduction, developer time, and measurable business outcomes |
| Allocation | Shared-cost rules, tags/labels, account/cluster/namespace/workload dimensions, discounts and commitment treatment |
| Sensitivity | Demand growth, log volume, cross-zone traffic, Spot interruption, utilization, support tier, dual-run duration, schedule variance |

Cost-reduction claims require matched periods, normalized business demand,
documented allocation, and retained source queries. Lower infrastructure cost is
not assumed merely because the target uses containers.

## Migration-wave scoring and selection

Score candidates consistently, but retain architect judgment and explain every
override.

| Dimension | Lower-risk signal | Higher-risk signal |
|---|---|---|
| Business criticality | Bounded internal capability | Safety, revenue, regulatory, or enterprise-wide path |
| Coupling | Observed, owned, versioned contracts | Hidden calls, shared runtime, undocumented file or operator dependency |
| Data | Stateless or compatible external store | Shared writes, large transfer, strict consistency, unclear ownership |
| Runtime | Horizontally scalable and supported | Singleton, host-bound, privileged, unsupported dependency |
| Network | Known flows and routable CIDRs | Overlap, fixed IP, low-latency hybrid, complex inspection or allowlists |
| Security/compliance | Standard controls and evidence | New control scope, unresolved exception, sensitive data without owner |
| Testability | Automated contract, load and journey tests | Manual validation, weak observability, no representative environment |
| Recoverability | Versioned rollback with compatible data | Irreversible change or untested recovery dependency |
| Operations | Named owner, SLO, telemetry and runbook | No support model, skill gap, hidden manual operation |
| Capacity | Measured profile within approved reserves | Unknown peak, quota request pending, no failure-state capacity |

The first wave should be useful enough to exercise the platform, delivery,
identity, observability, dependency, and rollback model, but bounded enough to
recover safely. A toy service proves too little; the most critical and coupled
service concentrates too much first-wave risk.

## Assessment deliverables

Before mobilization, retain versioned and owner-approved copies of:

1. business objectives, KPIs, constraints, governance, and decision authorities;
2. reconciled portfolio and infrastructure inventory with data quality scores;
3. application, data, identity, network, and operational dependency maps;
4. per-workload assessment records and migration-strategy decisions;
5. account, cluster, tenant, Region, and environment boundary decision;
6. Pod/node/IP/storage/control-plane demand model for normal, peak, release,
   maintenance, zone-failure, and recovery scenarios;
7. per-AZ subnet/IP worksheet and CNI/IP-family decision;
8. quota and throttle register with approved increases and fallbacks;
9. security, compliance, threat, exception, and evidence-ownership matrix;
10. SLO, RTO/RPO, backup, restore, resilience, cutover, and rollback contracts;
11. platform, pipeline, observability, support, lifecycle, and skills gap
    backlog;
12. baseline and target TCO model with allocation and sensitivity assumptions;
13. dependency-aware wave plan with scores, gates, owners, dates, and
    confidence; and
14. assumption, decision, risk, issue, and action registers with expiry or
    review dates.

## Exit criteria
Assessment is complete for a production wave only when:

- the workload and all critical dependencies have accountable owners;
- inventory fields used by architecture and planning have a source, collection
  date, confidence, and reviewer;
- tested per-replica capacity supports the peak, release, maintenance, selected
  failure, and recovery scenarios;
- node capacity, kubelet `maxPods`, CNI mode, Pod IP demand, per-AZ subnet
  space, storage attachments, and downstream limits reconcile in one capacity
  model;
- required AWS quota increases are approved and visible in the target account
  and Region;
- account, cluster, tenant, network, identity, data, and compliance boundaries
  have approved decisions;
- all release-blocking security findings and exceptions have disposition;
- SLO, RTO/RPO, observability, load, resilience, restore, cutover, abort, and
  rollback evidence requirements are agreed;
- the target landing-zone and platform prerequisites have owners and delivery
  dates earlier than the wave dependency date;
- the business case includes transformation, dual-run, operating, and
  decommission costs with documented uncertainty;
- operators have an enablement and acceptance plan; and
- every unresolved assumption has an owner, decision deadline, affected wave,
  and safe fallback.

If any of these conditions is unmet, the application may continue through
discovery or non-production experimentation, but it is not approved for a
production migration wave.

## Interview-ready reasoning

An enterprise assessment should be explainable as a chain of decisions:

1. Start with business criticality and real demand, not a preferred EKS design.
2. Translate observed demand into tested replica and resource requirements.
3. Test concurrent peak, release, maintenance, zone-loss, and recovery states.
4. Reconcile that Pod demand with nodes, VPC CNI addresses, per-AZ subnets,
   storage, load balancers, downstream services, control-plane churn, and
   quotas.
5. Select account, cluster, tenancy, network, identity, and lifecycle boundaries
   according to trust, blast radius, regulation, scale, and operating ownership.
6. Convert every material assumption into evidence, an owner, a deadline, and a
   gate before production traffic.
7. Reassess later waves using measured outcomes from earlier waves.

The key distinction is that application replica sizing answers how much runtime
is needed, while platform capacity assessment answers whether EKS and every
dependent AWS layer can create, address, schedule, connect, observe, and recover
that runtime safely.

## References

- [AWS Prescriptive Guidance: Application portfolio assessment](https://docs.aws.amazon.com/prescriptive-guidance/latest/application-portfolio-assessment-guide/introduction.html)
- [AWS Prescriptive Guidance: Evaluating migration readiness](https://docs.aws.amazon.com/prescriptive-guidance/latest/evaluating-migration-readiness/introduction.html)
- [Amazon EKS scalability best practices](https://docs.aws.amazon.com/eks/latest/best-practices/scalability.html)
- [Amazon EKS networking best practices](https://docs.aws.amazon.com/eks/latest/best-practices/networking.html)
- [Amazon EKS: Optimizing IP address utilization](https://docs.aws.amazon.com/eks/latest/best-practices/ip-opt.html)
- [Amazon EKS: Prefix mode for Linux](https://docs.aws.amazon.com/eks/latest/best-practices/prefix-mode-linux.html)
- [Amazon EKS control-plane scaling](https://docs.aws.amazon.com/eks/latest/best-practices/scale-control-plane.html)
- [Amazon EKS known limits and service quotas](https://docs.aws.amazon.com/eks/latest/best-practices/known_limits_and_service_quotas.html)
- [Amazon EKS multi-account strategy](https://docs.aws.amazon.com/eks/latest/best-practices/multi-account-strategy.html)
- [Amazon EKS tenant isolation](https://docs.aws.amazon.com/eks/latest/best-practices/tenant-isolation.html)
- [Amazon EKS reliability best practices](https://docs.aws.amazon.com/eks/latest/best-practices/reliability.html)
