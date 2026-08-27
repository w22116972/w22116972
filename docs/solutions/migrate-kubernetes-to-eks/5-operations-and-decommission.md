# Operations and Source Decommission

## Abstract

This phase defines the operating capability the customer must own before the
source cluster can be retired: the EKS fleet service catalog, observability
hierarchy, minimum runbooks, lifecycle and upgrade engineering, and knowledge
transfer. It closes with source-retirement readiness and the decommission
sequence.

## Operating outcome

Migration is complete when the customer can operate the EKS fleet and migrated
workloads independently, recover them from declared sources and protected data,
and retire the source cluster without an unknown traffic, state, identity,
security, operational, or rollback dependency.

EKS reduces self-managed control-plane responsibility. It does not remove
customer ownership of applications, data, nodes or selected compute mode,
cluster add-ons, access, policy, observability, backup, capacity, quotas, cost,
and Kubernetes lifecycle.

## Responsibility model

| Capability | Enterprise/cloud team | EKS platform team | Application/data team | Security | Operations |
|---|---|---|---|---|---|
| Account, network, IPAM, DNS and enterprise services | A/R | C | I | C | C |
| EKS cluster, nodes, capacity and core add-ons | C | A/R | C | C | R |
| CRDs/operators and cluster-scoped application services | I | R | A/R by ownership | C | C |
| Application desired state, image, configuration and SLO | I | C | A/R | C | R |
| Persistent data, backup and recovery validation | C | C | A/R | C | R |
| Human/workload identity and secrets policy | C | R | R | A | C |
| Admission, Pod and runtime security | C | R | R | A | C |
| Cutover and rollback decision | C | R | A/R | C | R |
| Incident command and communication | I | R | R | C | A |
| Source freeze, archive and decommission | R | R | A by workload | C | R |
| Cost allocation and optimization | A | R | R | I | C |

`A` is accountable, `R` responsible, `C` consulted, and `I` informed. Private
service records contain names and escalation routes; public examples remain
symbolic.

## EKS fleet service catalog

Every cluster cell records:

- business purpose, environments, tenants, AWS account, Region and owner;
- Kubernetes version, support window, endpoint model and cluster configuration;
- VPC/subnets, IP family, CNI mode, Service CIDR, DNS and connectivity
  ownership;
- node/compute modes, pools, architectures, capacity, quotas and scaling owner;
- EKS-managed and third-party add-ons with source, version, compatibility,
  lifecycle, configuration and rollback;
- ingress/Gateway, certificates, external/internal DNS and traffic ownership;
- CSI/StorageClasses, data services, backup and recovery ownership;
- human authorization, workload identity, secret and key integration;
- admission, Pod security, policy, runtime and audit controls;
- application namespaces, products, criticality, SLOs, dashboards and on-call;
- Terraform/GitOps/Helm/pipeline writers and effective revision evidence; and
- cost allocation, maintenance, upgrade, recovery and retirement schedule.

Unowned cluster-scoped APIs, controllers, webhooks, StorageClasses,
GatewayClasses or cluster roles block handoff.

## Observability hierarchy

```text
enterprise journeys and business output
  -> service objectives, traffic cohorts and migration waves
    -> application rate, errors, duration, queues and dependencies
      -> Kubernetes rollout, readiness, scheduling and autoscaling
        -> nodes, CNI/IP, DNS, ingress/Gateway, CSI/storage and controllers
          -> AWS APIs, quotas, identity, network, audit and cost
```

Required operational signals include:

- customer journeys, valid demand, latency distributions, errors and SLO burn;
- source versus target traffic until source retirement;
- Pods, Deployments, StatefulSets, Jobs, HPA, disruption, pending, eviction,
  restart, throttling and OOM state;
- node readiness, pressure, capacity, interruptions, replacements and zone
  distribution;
- CNI allocation, subnet IPs, ENIs, DNS QPS/errors, conntrack, bandwidth,
  load-balancer targets and TLS/certificate expiry;
- CSI/controller state, volumes, attachment, capacity, latency, throughput,
  backup and restore;
- API server latency/errors, controller work queues, webhooks, object/etcd
  growth, events and client throttling;
- IAM/authorization failures, policy denials, secret/certificate rotation and
  audit/security events;
- image, chart/Git revision, add-on and effective runtime identity; and
- account, cluster, namespace, workload, storage, network, observability, backup
  and dual-run cost.

Alerts map to an owner, service impact, safe first action, runbook and test. A
large collection of platform metrics without customer impact and ownership is
not operational readiness.

## Minimum runbooks

| Runbook | Required decision content |
|---|---|
| Application deploy/rollback | Source-to-running identity, health gates, traffic/data effects, last safe revision |
| Cluster access failure | Federation, EKS endpoint/DNS, authorization, break-glass and audit |
| Pod pending or scale failure | Requests, topology, taints, node pools, `maxPods`, CNI/IP, volume and quotas |
| CNI/DNS/network incident | Scope, recent change, subnet/IP, ENI, policy, DNS, routing, load balancer and rollback |
| Storage/data incident | CSI/attachment, topology, application writer safety, backup/restore owner and integrity proof |
| Controller/webhook failure | API/CRD ownership, endpoints/certificates, failure policy, dependent resources and safe bypass |
| Identity/secret/certificate failure | Effective ServiceAccount/role/reference, rotation, revocation, no value output |
| Node or zone disruption | PDB/topology, stateful protection, capacity reserve, reschedule and traffic behavior |
| Cluster/add-on upgrade | Compatibility matrix, deprecated APIs, rehearsal, order, drain, rollback and evidence |
| Backup and recovery | Configuration source, data restore order, identity/network prerequisites, application and traffic proof |
| Capacity/quota exhaustion | Demand, pool/AZ constraint, CNI/IP, AWS quota, downstream limit and approved expansion |
| Source-cluster rollback | Traffic route, source capacity/configuration, writer authority, data direction and expiry |

## Lifecycle and upgrade engineering

The fleet maintains a version matrix for Kubernetes, nodes, EKS add-ons,
controllers, CRDs, webhooks, clients, charts and APIs. The process:

1. monitors EKS and upstream lifecycle announcements;
2. inventories deprecated API use in declared and effective state;
3. tests target versions and add-ons in representative non-production cells;
4. validates controllers, webhooks, policy, CNI, CSI, DNS, ingress/Gateway,
   identity, observability and applications;
5. confirms subnet IPs, quotas and replacement capacity;
6. upgrades one minor version and layer at a time according to supported paths;
7. replaces/drains nodes within disruption and service objectives;
8. validates customer journeys, data and recovery; and
9. records effective versions and removes temporary compatibility.

Rollback is version- and component-specific. A managed control-plane rollback
does not automatically roll back every node, add-on, CRD, controller or
application change; the runbook defines all customer-owned layers.

## Knowledge transfer and acceptance

1. **Architecture teach-back:** operators explain account, cluster, request,
   identity, network, storage, data, delivery and recovery paths.
2. **Guided platform operation:** delivery team demonstrates access, deployment,
   diagnosis, scaling, node replacement, add-on health and evidence capture.
3. **Paired wave:** customer operator executes while the delivery team can
   intervene.
4. **Failure exercise:** customer handles a safe workload, identity, network,
   storage or controller failure.
5. **Independent change:** customer deploys and rolls back a bounded workload
   change through the approved owner.
6. **Independent recovery:** customer rebuilds required configuration, restores
   data and proves application traffic.
7. **Lifecycle rehearsal:** customer assesses and executes a representative
   add-on/node/version change in non-production.
8. **Support transition:** open gaps have owners/dates and normal
   incident/vendor escalation replaces project dependency.

Attendance is not acceptance. Evidence records correct decisions, safe
execution, recovery proof and unresolved actions.

## Source-retirement readiness

The source cluster enters a change freeze before decommission, except for
approved security or recovery changes. Retirement requires proof that:

- no production or internal traffic reaches source Services, Ingresses,
  Gateways, nodes or endpoints;
- no producer, consumer, scheduler, webhook, operator, integration, partner,
  allowlist, DNS record or certificate depends on the source;
- all authoritative data and retained history are migrated, reconciled,
  protected and recoverable;
- source queues, CronJobs, Jobs and singleton writers are stopped or retired;
- target objectives and recovery pass through the required observation period;
- source rollback has formally expired or been replaced by target recovery;
- required manifests, charts, images, audit, reports, configuration and evidence
  are archived under policy;
- secret, certificate, identity, registry and external-system access are revoked
  or rotated safely;
- monitoring, alerts, backups, licenses, support, network routes, load
  balancers, DNS and cost allocations are removed or reassigned; and
- application, platform, data, security, operations and business owners approve
  retirement.

## Decommission sequence

1. record final source inventory, traffic, data, backup, access and evidence;
2. stop remaining scheduled/async work and confirm target ownership;
3. remove source from traffic and service discovery, then observe for hidden
   use;
4. revoke source workload and human access in dependency order;
5. archive or expire required logs, audit, manifests, images and backups;
6. release external integrations, certificates, DNS, load balancers, network
   routes, storage and licenses only after owner confirmation;
7. delete source workloads and cluster through the approved platform procedure;
8. verify infrastructure, monitoring, backup, billing and CMDB cleanup;
9. close dual-run cost and record residual retained artifacts; and
10. hold a retrospective and feed findings into remaining cluster migrations.

Deletion is deliberately last. A quiet source cluster is not proof that no
hidden consumer, scheduled workload or recovery dependency remains.

## Operational review cadence

### Per release

- reconcile source/chart/image/configuration/running identity;
- evaluate service objectives, dependencies, policy and rollback;
- record exceptions and evidence.

### Weekly during migration

- review wave gates, source/target drift, traffic, data lag, incidents,
  capacity, quotas, retirement blockers and dual-run cost;
- select later waves from evidence rather than schedule pressure.

### Monthly after migration

- review SLOs, incidents, alerts, vulnerabilities, Kubernetes/add-on lifecycle,
  certificates, secrets, backup/restore, capacity, quotas, cost and ownership;
- exercise failure and recovery on the agreed cadence;
- remove expired exceptions and temporary coexistence paths.

## Handoff package

The private package contains:

- requirements, inventory, compatibility and architecture decisions;
- source/target ownership and drift reconciliation;
- infrastructure, platform and application desired-state locations/revisions;
- migration reports with approved skipped/replaced resources;
- traffic, data, integrity, rollback and observation evidence;
- service catalog, RACI, SLOs, dashboards, alerts and escalation;
- deploy, rollback, incident, capacity, identity, network, storage, upgrade,
  recovery and decommission runbooks;
- quota, lifecycle, risk, exception, debt and cost registers; and
- source-retirement approvals and retained recovery artifacts.

The public portfolio omits real identifiers, manifests, reports, credentials,
addresses and customer data.

## Completion and claim boundary

This reference solution is complete as a design when its lifecycle and evidence
requirements are usable. A real migration is complete only when target traffic,
data, recovery, operations and source decommission pass. Until then, do not
claim migrated workload count, downtime, savings, availability or reduced
operational effort.

## References

- [AWS Kubernetes-to-EKS migration best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/best-practices.html)
- [Amazon EKS cluster lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html)
- [Amazon EKS cluster upgrade guidance](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)
