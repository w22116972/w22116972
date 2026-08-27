# Cutover, Validation, and Results

## Abstract

This phase validates along the customer request and data path first, then
descends into Kubernetes and AWS layers to explain failures. It defines the
evidence packet, pre-cutover gates, cutover sequence, abort criteria, rollback
models, and the claim boundary that stops object counts and Pod readiness from
being reported as migration outcomes.

## Validation principle

Validation follows the customer request and data path, then descends into
Kubernetes and AWS layers to explain failures. Object counts and Pod readiness
are necessary reconciliation checks, not migration outcomes.

Evidence is separated into gates:

```text
declared source and approved transformation
  -> target API and policy acceptance
    -> controller reconciliation
      -> healthy workload and platform runtime
        -> dependency and data behavior
          -> real traffic and service objectives
            -> recovery and independent operation
```

## Evidence packet

Every wave packet records:

- source cluster, target cluster cell and symbolic wave ID;
- owners, approvers, decision authority and escalation;
- source Git/chart/manifest revision, rendered target revision and
  transformation version;
- image digests, target Kubernetes/platform/add-on versions and data checkpoint;
- expected, created, reconciled, replaced, retained, skipped and error counts;
- baseline and target queries, thresholds, evaluation windows and dashboards;
- data synchronization, integrity and writer-authority state;
- cutover unit, schedule, communication and observation period;
- abort criteria, rollback steps and last reversible point; and
- evidence locations, timestamps, reviewers, exceptions and next decision.

## Pre-cutover gates

### 1. Desired-state and ownership gate

- every target resource has an authoritative writer;
- live source drift has an approved disposition;
- removed/generated fields and system resources are excluded;
- APIs, schemas, render, policy and server-side dry run pass;
- expected skipped/replaced resources are reviewed rather than hidden; and
- source, chart, image and target desired-state revisions are pinned.

### 2. Platform reconciliation gate

- nodes are Ready across required zones and node pools;
- CNI, DNS, proxy, CSI, ingress/Gateway, identity, secrets, certificates,
  policy, observability and backup components are healthy and owned;
- CRDs are served and controllers/webhooks reconcile without error;
- required allowed actions and denied actions behave correctly;
- IP, node, storage, target, API and quota capacity is available for cutover,
  surge, rollback and selected failures; and
- target telemetry and audit evidence are complete enough to judge the wave.

### 3. Workload gate

- desired/available replicas match and scheduling distribution is expected;
- startup, readiness, liveness, graceful termination and disruption behavior
  pass;
- source and target image/configuration/secret identities are understood;
- Services, endpoints, DNS, ingress/Gateway and dependencies work through the
  intended request path;
- resource, autoscaling, connections, queues and platform API behavior remain
  inside approved thresholds under representative load; and
- security, contract, integration, synthetic and failure tests pass.

### 4. Data gate

- initial and final synchronization checkpoints are identified;
- lag, volume, counts, checksums, schemas, ordering, duplicates and
  application-specific invariants pass;
- target read/write behavior is tested through the application;
- backup and restore produce usable application state;
- writer authority, fencing and split-brain protection are clear; and
- rollback direction remains supported at the planned traffic step.

### 5. People and change gate

- source, target, application, data, network, security and incident owners are
  present or on call;
- runbook, commands, dashboards and communication templates are reviewed;
- change freezes and concurrent releases are controlled;
- operators have executed the rehearsal and rollback; and
- the accountable release owner explicitly authorizes traffic movement.

## Cutover sequence

1. Announce the start and verify no unapproved source, target, data or delivery
   changes are active.
2. Record source and target health, traffic, capacity, data checkpoint and
   effective release identities.
3. Confirm source capacity and configuration can accept rollback traffic.
4. Start or verify final data synchronization; quiesce/fence writers if
   required.
5. Deploy or scale the target wave and confirm full controller reconciliation.
6. Run internal request-path, identity, secret, dependency and data probes.
7. Move the smallest approved traffic unit.
8. Measure actual source/target request distribution and evaluate fast-failure
   gates.
9. Observe for the approved interval, including latency/error distributions,
   saturation, queues, dependencies and data invariants.
10. Expand one bounded increment only after explicit approval.
11. When target writer authority changes, record the last reversible point and
    enforce the data rollback contract.
12. Hold the final wave through representative demand before declaring cutover
    complete.

Traffic methods can include weighted DNS, load-balancer/gateway weight, route,
tenant, customer cohort or scheduled consumer switch. The chosen method must
support the protocol and client behavior; DNS weights alone do not guarantee
precise traffic because of resolver caching and persistent connections.

## Abort criteria

Stop expansion and execute the applicable rollback when:

- critical user journey, availability, latency or error objective breaches its
  threshold and evaluation period;
- target traffic distribution is unknown or materially differs from the planned
  step;
- data lag, integrity, ordering, duplicate, reconciliation or writer-authority
  checks fail;
- dependencies, queues, databases, certificates, identity, secrets or external
  allowlists fail or saturate;
- Pods crash, OOM, throttle, remain Pending, lose topology, or scale unstably;
- CNI/IP, DNS, load balancer, storage, node, API or quota capacity approaches
  the approved reserve;
- source-to-target release identity cannot be reconciled;
- observability or audit evidence is missing; or
- source rollback capacity, route, credentials, data direction or owner is no
  longer available.

Numeric values live in the private wave packet. This public reference does not
invent universal latency, error, downtime or observation thresholds.

## Rollback models

### Stateless or read-only wave

1. freeze traffic expansion and target configuration;
2. restore route/weight/DNS to the source;
3. confirm real source request paths and objectives;
4. scale down or isolate target only after connections drain and evidence is
   preserved; and
5. investigate and add a new test before reopening the wave.

### Stateful wave before writer transfer

Restore source traffic, stop target consumers/writers, preserve the failed
target state for diagnosis, and continue or restart synchronization only through
the approved data procedure.

### Stateful wave after writer transfer

Rollback is system-specific and may not mean immediate source activation:

1. stop unsafe writers and traffic expansion;
2. determine the authoritative dataset and last consistent checkpoint;
3. execute tested reverse replication, replay, restore or forward-fix decision;
4. validate integrity and application behavior;
5. route traffic only to the environment authorized to write; and
6. reconcile delayed/duplicate transactions before normal operation.

If reverse data movement is not supported, document that the last reversible
point precedes writer transfer. Do not advertise source-cluster rollback after
that point without a tested data mechanism.

## Validation matrix

| Layer | Test | Evidence |
|---|---|---|
| Inventory | Expected/migrated/replaced/retained/skipped/error reconciliation | Signed wave report and owner disposition |
| API | Supported APIs, schemas, server-side dry run and admission | Rendered artifacts and target API results |
| Controllers | CRD/operator/webhook conditions and external effects | Controller metrics/events plus resource conditions |
| Scheduling | Requests, topology, taints, affinity, PDB and node/AZ failure | Placement and failure exercise |
| Network | DNS, Service, ingress/Gateway, source IP, TLS, policies, egress and hybrid paths | Allowed/denied probes and flow evidence |
| Storage | Provision, attach, mount, topology, performance and failure | CSI/volume state plus application I/O |
| Identity/security | Human/workload allowed and denied actions, policy and audit | Test identities and audit records |
| Application | Contracts, user journeys, functional parity and dependency failures | Automated results and trace correlation |
| Performance | Matched demand, latency, errors, saturation, scale and burst | Source/target time series and load report |
| Data | Counts/checksums/schema/order/duplicates/reconciliation/read-write | Data-owner-approved report |
| Recovery | Configuration rebuild, data restore, controller reconcile and traffic | Timed exercise and application proof |
| Operations | Deploy, diagnose, scale, rollback and incident exercise | Operator scorecard and open actions |

## Post-cutover observation

Compare source baseline and target over representative business cycles:

- successful business output and demand composition;
- p50/p95/p99 latency, errors and availability against service objectives;
- CPU, memory, throttling, restarts, scheduling, autoscaling and node behavior;
- CNI/IP, DNS, load balancer, storage, connection and network behavior;
- database, queue, cache, external dependency and data synchronization health;
- alerts, incidents, operator interventions and support burden;
- API/controller/webhook churn and cluster-service saturation; and
- source, target and dual-run costs under the same allocation rules.

The observation window is extended when seasonality, batch schedules,
certificate rotation, backup, node maintenance, autoscaling, or external
dependency cycles have not yet been exercised.

## Measurement contract

| Outcome | Calculation or comparison |
|---|---|
| Completeness | approved migrated and replaced resources / approved in-scope resources, with skipped/error disposition |
| Functional parity | critical contracts and journeys passed on target / required contracts and journeys |
| Performance | target indicator distribution versus matched source demand and payload |
| Availability | successful valid target requests / valid target requests during approved window |
| Recovery | measured configuration rebuild + data restore + reconcile + application/traffic validation time |
| Operational effort | comparable platform maintenance/incident/change effort with scope and period |
| Unit cost | eligible platform and workload cost / successful business output |

Reports retain dates, environments, demand, inclusion rules, source queries,
allocations, discounts, confounders, authors and reviewers. Object counts alone
do not prove customer value.

## Claim boundary and result template

Until a real engagement provides retained evidence, use this statement:

> Designed an enterprise migration pattern for moving self-managed Kubernetes
> workloads to Amazon EKS through source-of-truth reconciliation, API and
> platform compatibility mapping, dependency-ordered deployment,
> application-consistent state migration, progressive traffic cutover, tested
> rollback, and source decommission gates. No production scale, downtime,
> cost, or availability result is claimed without an attached evidence packet.

Do not convert the AWS sample tooling's completed report, zero API error count,
or Ready Pod count into a completed migration claim.

## Exit criteria
A wave exits only after full traffic and writer authority are understood; target
service objectives and data invariants pass through the observation period;
source rollback dependency and last reversible point are recorded; recovery and
operator exercises pass; evidence is reviewed; and remaining risks have owners.

## References

- [AWS Kubernetes-to-EKS migration procedures](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/migration-procedures.html)
- [AWS Kubernetes-to-EKS best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/migrating-self-managed-kubernetes-cluster-to-amazon-eks/best-practices.html)
- [AWS migration pre-cutover guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/best-practices-migration-cutover/pre-cutover-stage.html)
