# Migration Implementation

## Abstract

Migrate a dependency-complete workload wave into a separately built EKS cluster.
Do not mutate the source into EKS, copy its control-plane state, or treat every
live Kubernetes object as portable desired state.

The implementation keeps these control loops separate:

```text
source inventory -> evidence and reconciliation
declared source   -> reviewed target desired state
Terraform         -> AWS and EKS foundation
platform delivery -> cluster-scoped services and APIs
application owner -> namespace/workload resources
data workflow     -> application-consistent state
traffic owner     -> cutover and rollback
```

## Execution phases

| Phase | Input | Change | Validation | Rollback |
|---|---|---|---|---|
| 1. Freeze wave definition | Approved assessment and dependency group | Freeze scope, owners, sources, versions, thresholds and evidence locations | Wave packet review | No runtime change |
| 2. Build target cell | Architecture and capacity decisions | Provision AWS/EKS foundation and platform services declaratively | IaC plan/apply plus AWS, cluster, node, add-on, identity, network, storage and audit readback | Revert bounded foundation change or rebuild before workloads |
| 3. Reconcile desired state | Live inventory plus Git/Helm/Kustomize/operator sources | Remove generated fields, convert APIs, map integrations, resolve drift and writers | Render, schema, policy, server-side dry run, ownership review | Source remains unchanged |
| 4. Rehearse platform dependencies | Approved CRDs/operators and application prerequisites | Install and test platform/application controllers in dependency order | API discovery, webhook, reconcile, allowed/denied and failure tests | Remove bounded target components or rebuild target |
| 5. Rehearse workload and data | First-wave declarations and data method | Deploy in non-production, migrate representative state, exercise traffic and recovery | Functional, load, security, failure, integrity, restore and rollback evidence | Delete/rebuild rehearsal target; source unaffected |
| 6. Prepare production | Passed rehearsal and available quotas | Create target resources, secret references, inactive data sync and observability | Preflight, capacity, freshness and operator readiness | Stop before writer/traffic change |
| 7. Synchronize and deploy | Approved cutover packet | Synchronize/quiesce state, deploy workloads and reconcile controllers | Runtime identity, dependency, data, SLO and source/target comparison | Stop target writers and restore source authority per data contract |
| 8. Shift traffic | Healthy target and executable rollback | Move bounded route, tenant, cohort or weight | Real request paths, objectives, dependencies and data invariants | Restore traffic and source writer authority |
| 9. Observe and expand | Passed bounded wave | Increase traffic or proceed to next dependency-complete wave | Observation across representative demand and failure conditions | Roll back current reversible wave |
| 10. Transfer and retire | All waves stable and source dependencies cleared | Handoff, freeze/archive and decommission source | Independent operation, recovery, evidence and cost closure | Retain approved recovery artifacts; source recreation only if designed |

## Safe source collection

Collection uses an explicitly named read-only context and least-privilege
credentials. Operators verify the active context before every source or target
action. Source extraction and target application never run from an ambiguous
default context.

Capture structured summaries for:

- cluster and node versions, capacity, labels, taints, runtime and architecture;
- API resources, CRDs, custom resources and admission webhooks;
- namespaces, Deployments, StatefulSets, DaemonSets, Jobs and CronJobs;
- Services, Ingresses, Gateway resources, EndpointSlices and NetworkPolicies;
- StorageClasses, PV/PVC metadata, volume snapshots and CSI drivers;
- ServiceAccounts, Roles, RoleBindings, ClusterRoles and ClusterRoleBindings;
- ResourceQuotas, LimitRanges, PodDisruptionBudgets, priority and scheduling
  policies;
- ConfigMaps and Secret metadata only;
- Helm release metadata and GitOps application ownership;
- image references/digests, probes, resources, security contexts and
  dependencies; and
- controller status, events, telemetry and observed network flows.

Do not export Secret values or blindly archive complete namespaces. Treat the
collection as sensitive, encrypt it, restrict access, record its collection time
and source revision, and delete it according to the engagement retention policy.

## Desired-state transformation

### Remove non-portable server state

From declarations intended for the target, remove or regenerate:

- `status`;
- UIDs, `resourceVersion`, generations, timestamps and managed fields;
- owner references whose owner is regenerated in the target;
- assigned nodes, Pod IPs, host IPs, ClusterIPs and provider addresses;
- default service-account token Secrets and other generated credentials;
- source load-balancer status and provider finalizers/annotations;
- source PV binding details unless the data migration design explicitly creates
  compatible target resources;
- controller-generated Pods, ReplicaSets, EndpointSlices and children; and
- source platform namespaces, roles, admission, CNI, CSI, DNS and control-plane
  objects owned by the target platform.

This filtering does not make an arbitrary export safe. The target declaration
still requires a known writer, review, target API conversion and ownership.

### Convert APIs and integrations

For each target declaration:

1. render the authoritative chart/overlay at its pinned source revision;
2. convert removed/deprecated APIs and review semantic differences;
3. replace source-specific storage, ingress, load-balancer, identity, secret,
   policy, DNS and node annotations;
4. resolve target namespace, labels, quotas, priority, topology and workload
   identity;
5. pin or approve image digests and registry paths;
6. validate schemas, CRDs and admission policies;
7. perform a server-side dry run against the actual target API; and
8. inspect the final rendered diff before promotion.

Text substitution is not sufficient for API migration. Defaults, selectors,
webhook mutation, policy, controller behavior and field semantics must be
tested.

## Platform and CRD deployment order

Use dependency ordering rather than one bulk apply:

```text
namespaces and baseline tenancy
  -> platform CRDs and APIs
    -> platform controllers and webhooks
      -> storage, network, identity, secret and certificate integrations
        -> application CRDs
          -> application operators
            -> policy and configuration prerequisites
              -> application custom resources and workloads
                -> Services, ingress or Gateway routes
                  -> traffic
```

After each layer:

- confirm the expected API is served;
- confirm controller replicas and leader election;
- verify webhook endpoint/certificate and failure behavior;
- inspect conditions and events for successful reconciliation;
- test required and denied external effects; and
- stop if an owning controller is missing or two writers claim the same object.

CRD installation success is not operator readiness. Custom-resource creation
success is not proof that its cloud, network, storage or application effects
exist.

## Identity, RBAC, policy, and secrets

### Human and workload identity

- map approved enterprise personas to federated AWS roles and EKS authorization;
- create job-function RBAC from least privilege rather than importing broad
  source groups;
- give each workload a dedicated ServiceAccount;
- map required AWS API actions to EKS Pod Identity or IRSA as appropriate;
- remove static and node-wide application credentials; and
- test successful required actions and failed unauthorized actions.

### Policy migration

- translate source Pod security intent to Pod Security Admission and/or the
  approved target policy engine;
- remediate privileged/root/host namespace/hostPath/capability/seccomp and
  writable-filesystem dependencies;
- port network policy only after verifying target CNI enforcement semantics;
- scope admission policies to avoid blocking their own recovery dependencies;
- test webhook unavailability, timeout and emergency bypass; and
- give exceptions owners, compensating controls and expiry dates.

### Secret and certificate migration

Repositories and extraction output contain references and metadata, never
values. Before workload activation:

1. create approved target secret/certificate authority mappings;
2. authorize the target ServiceAccount or controller;
3. populate or synchronize the target through the secret workflow;
4. confirm ownership, rotation, reload and revocation;
5. test the application without printing values; and
6. revoke obsolete source access after rollback dependencies end.

## Networking and service exposure

Rebuild networking from required behavior:

- create Services without copying allocated ClusterIPs;
- map Ingress/Gateway/controller annotations to the selected target data plane;
- verify protocols, TLS termination, mTLS, client/source IP, headers, health
  checks, idle timeouts, long-lived connections and WebSockets where applicable;
- map internal/external load balancers, security groups and target registration;
- validate CoreDNS, enterprise DNS/Resolver, stub domains and dependency names;
- validate allowed and denied NetworkPolicy paths;
- validate target egress IP, proxies, inspection, partner allowlists and NAT;
- compare MTU, fragmentation, connection tracking, latency and bandwidth; and
- monitor target health before publishing DNS or route changes.

For weighted DNS, TTL is not the only cache lifetime: clients, resolvers,
connection pools and long-lived sessions can delay effective traffic movement.
Measure actual source and target request distribution.

## Images and registry

- reconcile every running source image to an immutable digest and build/source
  identity where available;
- decide whether to retain an approved existing registry, copy images to ECR, or
  rebuild through the target supply chain;
- scan and disposition the exact promoted digest;
- test target architecture and OS compatibility;
- preserve required signatures/provenance and SBOM relationships;
- test pull permissions, private endpoints/proxies, rate and concurrent node
  bootstrap demand; and
- retain required source images until rollback and retention windows expire.

Copying an image does not repair an unknown build or unsupported dependency.
Untraceable critical images require a risk decision and remediation plan.

## Stateful workload implementation

Select a method per data system:

| Pattern | Use when | Required controls |
|---|---|---|
| Backup and restore | Downtime and RPO permit a point-in-time move | Quiesce, application-consistent backup, transfer, restore, checksum/invariants, write test |
| Continuous replication | Low downtime and supported replication exist | Initial sync, lag monitoring, encryption, consistency, cutover/failback and split-brain prevention |
| Application dual-write | Application owns tested idempotent compatibility | Ordering, duplicates, reconciliation, failure handling and removal plan |
| Export/import | Dataset and window are bounded | Version compatibility, encryption, bandwidth, completeness, sequence and retry |
| Recreate disposable state | Cache/index can be safely rebuilt | Authoritative source, rebuild capacity/time, consistency and dependency protection |

The runbook defines writer authority through every step:

```text
source writer authoritative
  -> initial target synchronization
    -> catch up and validate
      -> quiesce or fence as required
        -> final synchronization and integrity gate
          -> target writer authoritative
            -> observe before ending source rollback support
```

PersistentVolume and PVC manifests are deployed only after the target storage
and data method exist. A Bound PVC proves attachment, not data correctness.

## Rehearsal and wave promotion

At least one complete wave is rehearsed in a representative environment using
the same ordered automation and decision gates intended for production.

The rehearsal includes:

- extraction/inventory reconciliation and skipped-resource review;
- clean target rebuild from declared sources;
- CRD/operator/webhook ordering;
- identity, secret, policy, network and storage mappings;
- representative data transfer and integrity proof;
- load, scale, node/zone failure, dependency degradation and restore;
- cutover and rollback including DNS/traffic and writer authority;
- source/target dashboards and evidence capture; and
- operator-led execution from the runbook.

Promotion pins source revision, chart/manifest versions, image digests, target
platform versions, data checkpoint, evidence queries and approvers. Rehearsal
findings update later wave estimates and controls.

## Automation guardrails

Migration tooling should support inventory, normalization suggestions, ordered
deployment, dry run, reports and resumption, but automation does not own
architectural decisions.

Required guardrails include:

- explicit source and target context arguments and identity display;
- read-only source permissions except separately approved data mechanisms;
- default dry-run behavior for target Kubernetes changes;
- include lists per approved wave rather than full-cluster default;
- exclusion of Secrets values, generated children and system resources;
- deterministic rendered output and versioned transformation rules;
- expected, migrated, replaced, retained, skipped, warning and error counts;
- halt on unknown CRD owner, missing controller, policy denial, target mismatch,
  secret reference failure or data precondition;
- idempotent/resumable phases where safe; and
- encrypted reports, redacted logs, retention and reviewer sign-off.

An automation report showing zero API errors proves mechanism execution only.
Runtime, traffic, data and operations need separate gates.

## Exit criteria
A wave can request production traffic only when target desired state has one
owner; platform and application controllers reconcile; identities, secrets,
network, storage and data paths pass; capacity and quotas are available; source
and target telemetry is comparable; the rollback path is current; and operators
can execute the cutover packet without undocumented commands.

## References

- [AWS Prescriptive Guidance: Kubernetes-to-EKS migration procedures](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/migration-procedures.html)
- [AWS Prescriptive Guidance: Kubernetes-to-EKS best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/best-practices.html)
- [Amazon EKS create-cluster considerations](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)
