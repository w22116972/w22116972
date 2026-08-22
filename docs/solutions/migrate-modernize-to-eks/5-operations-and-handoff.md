# Operations and Handoff

## Operating outcome

Migration is complete when the customer team can operate, change, diagnose,
recover, and improve the EKS service estate without depending on the delivery
team. More services increase the need for clear ownership and common signals;
handoff is therefore an acceptance gate, not a final presentation.

The retained record does not include a historical handoff sign-off. This chapter
defines the operating model and evidence that a comparable delivery should
retain, without claiming that every artifact below was completed in the
historical program.

## Service ownership contract

Each service registers:

- business capability and product owner;
- technical and on-call owner;
- upstream callers and downstream dependencies;
- source, image, chart, runtime namespace, and dashboard;
- service objectives and alert routes;
- configuration and secret owners;
- data classification, backup, and recovery dependency;
- deployment, rollback, scaling, and incident runbooks; and
- known limitations, temporary adapters, and retirement criteria.

Unowned services do not enter production traffic.

## Service objectives and signals

| Signal | Example objective or question | Operator action |
|---|---|---|
| Availability | Are valid requests completing inside the agreed ratio? | Confirm blast radius, route, release, and dependencies |
| Latency | Are p50, p95, and p99 inside the workload objective? | Separate application, queue, database, network, and external latency |
| Errors | Are failures new, release-correlated, tenant-specific, or dependency-driven? | Compare version/cohort and trace the request path |
| Saturation | Are CPU, memory, threads, connections, queues, or storage near a safe boundary? | Apply the workload-specific scale or protection runbook |
| Kubernetes state | Are pods Ready, restarting, throttled, pending, or unevenly placed? | Correlate pod state with service behavior; do not stop at `kubectl get pods` |
| Dependency health | Is a downstream system slow, unavailable, unauthorized, or misconfigured? | Protect retry budgets, degrade deliberately, and engage the owner |
| Delivery state | Do source, image, chart, release, and running digest agree? | Stop change, identify the writer, and restore a known revision |
| Cost and usage | Is cost per business output changing, and why? | Attribute demand, resource, capacity, and pricing effects before acting |

Objectives use workload-approved numeric values. This public document does not
invent availability or latency thresholds that are absent from the retained
evidence.

## Dashboard hierarchy

```text
portfolio
├── customer journeys and service-objective burn
├── migration wave and release health
├── service RED signals: rate, errors, duration
├── resource USE signals: utilization, saturation, errors
├── Kubernetes: readiness, restarts, scheduling, HPA and nodes
├── dependencies: data, queues, cache and external systems
└── cost: allocated compute and cost per business output
```

The portfolio view answers whether customers are affected. Lower layers explain
why. Operators should not need to assemble a user-impact view from raw pod logs
during an incident.

## Minimum runbook set

| Runbook | Trigger | Safe first actions | Recovery proof |
|---|---|---|---|
| Deploy | Approved release | Verify source/image/chart identity and current health | New revision healthy through real traffic path |
| Rollback | Cutover or release threshold breach | Freeze change, restore traffic, deploy prior revision | User journey and data invariants restored |
| Pod not Ready | Readiness or rollout alarm | Inspect events, effective config, dependency, resource state, and previous logs | Expected replicas Ready and service objective recovered |
| Crash or OOM loop | Restart or memory alarm | Preserve previous logs, compare release/resource change, stop expansion | Stable process and memory behavior through observation window |
| Latency or errors | Service-objective burn | Split by version, route, tenant/cohort, and dependency | Error/latency distribution returns inside objective |
| HPA or capacity saturation | Pending pods, HPA maxed, or queue growth | Check request-relative threshold, node capacity, quotas, and dependency limit | Demand served without unstable scaling |
| Credential or authorization failure | Access-denied or auth alarm | Identify workload identity and secret reference; never print value | Required action succeeds and unauthorized action remains denied |
| Dependency outage | Timeout, queue, or circuit alarm | Bound retries, degrade safely, protect data consistency, engage owner | Recovery without replay, duplicate, or backlog loss |
| Node or zone disruption | Eviction, capacity, or placement alarm | Protect stateful workloads, observe PDB/topology, add approved capacity | Replicas and traffic recover within objective |
| Data recovery | Corruption or unavailable state | Stop unsafe writers and invoke system-specific restore plan | Integrity and application tests pass after restore |

Each runbook names prerequisites, commands or dashboards, decision authority,
escalation, rollback, evidence capture, and last exercise date. Examples use only
symbolic resources such as `cluster-a`, `namespace-a`, and `service-a`.

## Responsibility matrix

| Activity | Product owner | Service team | Platform team | Security | Operations |
|---|---|---|---|---|---|
| Approve behavior and migration wave | A | R | C | C | C |
| Build and test application release | C | A/R | C | C | I |
| Operate EKS, capacity, and add-ons | I | C | A/R | C | R |
| Own service objective and dashboard | A | R | C | I | R |
| Manage workload identity and secret policy | I | R | R | A | C |
| Execute production cutover | A | R | R | C | R |
| Declare rollback or incident | C | R | R | C | A |
| Approve legacy retirement | A | R | C | C | C |
| Validate allocated cost result | A | C | R | I | C |

`A` is accountable, `R` responsible, `C` consulted, and `I` informed. Names and
escalation channels belong in the private service catalog, not the public case
study.

## Knowledge-transfer sequence

1. **Architecture walkthrough** — customer operators explain the current and
   target request paths, ownership boundaries, and rollback dependencies back to
   the delivery team.
2. **Guided operation** — delivery engineer demonstrates deploy, observe,
   diagnose, rollback, and evidence capture in a non-production environment.
3. **Paired operation** — customer operator leads while the delivery engineer
   can intervene.
4. **Failure exercise** — inject a safe bad configuration, failed dependency,
   or unhealthy release and require diagnosis through service signals.
5. **Independent change** — customer operator releases and rolls back a bounded
   change without delivery-team commands.
6. **Independent incident** — customer team triages, communicates, recovers, and
   records evidence while the delivery team observes only.
7. **Sign-off and support transition** — open gaps receive owners and dates;
   normal support and escalation replace project dependency.

Training covers reasoning and decision boundaries, not memorizing a list of
commands.

## Independent-operation acceptance

| Capability | Pass condition | Evidence |
|---|---|---|
| Explain architecture | Operator traces an external request through EKS and its critical dependencies | Recorded review or assessed walkthrough |
| Release safely | Operator identifies current revision, deploys a bounded change, and proves effective state | Change record and verification output |
| Roll back | Operator restores traffic and the prior pinned release inside the approved target | Timed exercise and data check |
| Diagnose | Operator separates application, platform, configuration, dependency, and traffic failures | Scenario scorecard and timeline |
| Scale | Operator explains requests, HPA threshold, node capacity, and service limit before changing them | Reviewed capacity exercise |
| Handle secrets | Operator rotates or restores a credential without exposing it or changing unrelated workloads | Audit and allowed/denied test |
| Recover data | Operator locates the correct backup/restore owner and validates restored integrity | Restore exercise |
| Communicate | Operator states impact, scope, mitigation, owner, and next update | Incident simulation record |
| Improve | Team reviews service objectives, incidents, resource use, and cost on an agreed cadence | Review agenda and tracked action |

Completion requires all critical capabilities to pass or a risk owner to accept a
dated remediation. Attendance alone is not acceptance.

## Operational review cadence

### Per release

- validate source-to-runtime identity;
- watch service objectives and dependencies;
- record rollback availability and exceptions.

### Weekly during migration

- review wave evidence, incidents, error budget, unknown dependencies, and
  delayed retirement;
- compare capacity and workload behavior; and
- choose the next bounded wave based on readiness, not schedule pressure alone.

### Monthly after transition

- review objectives, alerts, runbook exercises, vulnerabilities, upgrades,
  backups, cost allocation, and ownership gaps;
- remove temporary compatibility paths when their exit conditions pass; and
- confirm that dashboards and escalation contacts still match the service
  estate.

## Handoff package

The final private package contains:

- sanitized architecture plus environment-specific private overlays;
- service catalog and responsibility matrix;
- infrastructure, image, chart, and pipeline ownership;
- dashboards, alert rules, objectives, and escalation routes;
- deploy, rollback, incident, capacity, credential, and recovery runbooks;
- migration-wave evidence and accepted exceptions;
- backup/restore dependencies and exercise results;
- resource and cost measurement workbook; and
- open risks, temporary adapters, debt owners, and retirement dates.

The public version omits account IDs, ARNs, repository addresses, internal
hosts, credentials, raw logs, and customer identifiers.

## Personal contribution and handoff boundary

The retained résumé supports an end-to-end leadership contribution across
containerization, Terraform, Helm, EKS, and the 20-plus-component modernization.
The implementation repositories also support hands-on work in image, chart,
pipeline, resource, policy, and operational patterns.

The public evidence does not support claiming sole delivery, a specific training
attendance count, a timed independent-operation result, or a formal AWS review.
Those claims remain excluded until a source artifact is available.

## Lessons for future engagements

- Design the ownership and handoff model while designing the cluster.
- Make real user behavior the top operational signal; Kubernetes status is a
  diagnostic layer beneath it.
- Exercise rollback before production traffic, including route and data effects.
- Treat temporary compatibility code as a governed migration asset with an exit
  condition.
- Normalize cost by business output and preserve the calculation.
- Retain the evidence packet; a strong result that cannot be reconstructed is a
  weak case study.

## Implemented extensions

- [Karpenter compute and elasticity modernization](5-2-karpenter-compute-and-elasticity-modernization.md)
- [Workload autoscaling modernization](5-3-workload-autoscaling-modernization.md)
- [Amazon EKS FinOps and cost optimization](5-4-eks-finops-and-cost-optimization.md)
- [Observability and SRE operating model](5-5-observability-and-sre-operating-model.md)
- [Amazon EKS lifecycle and upgrade engineering](5-6-eks-lifecycle-and-upgrade-engineering.md)
- [Platform engineering and golden paths](5-7-platform-engineering-and-golden-paths.md)
- [Agentic AIOps with Amazon Bedrock](5-8-agentic-aiops-with-amazon-bedrock.md)
- [Customer expansion and modernization roadmap](5-9-customer-expansion-and-modernization-roadmap.md)
