# Disaster Recovery of AWS Workloads

## Purpose

Design disaster recovery (DR) as a business-continuity capability, not as an
unverified collection of backups. A disaster is an event that prevents a
workload from meeting its business objectives at its primary location. The DR
plan must define how the complete workload - application, data,
infrastructure, configuration, access, and operations - will recover within
business-approved objectives.

High availability and DR reinforce each other but solve different problems.
Multi-AZ deployment, health checks, and automatic replacement help a workload
continue through common component or Availability Zone failures. DR addresses
larger events, including Regional disruption, destructive data changes, account
compromise, or an operational failure that makes the primary workload unusable.
Continuous replication alone is not DR for data corruption: it can replicate a
bad change. Keep tested point-in-time recovery as a separate protection.

## Start with business requirements

- Make DR a section of the Business Continuity Plan (BCP). Include upstream and
  downstream dependencies, people, facilities, suppliers, communications, and
  manual business processes; restoring an application is insufficient when the
  business cannot operate around it.
- Perform a business impact analysis for each workload and critical user
  journey. Record business owner, impact over time, legal or data-residency
  constraints, dependencies, acceptable degradation, and the cost of outage and
  data loss.
- Assess plausible threats by both type and geographic scope: natural events,
  technical failures, human error, malicious changes, and account compromise.
  State explicitly when a risk is accepted rather than silently leaving a
  workload without protection.
- Define measurable recovery objectives before selecting technology.
  - **RTO (Recovery Time Objective):** the maximum acceptable time from service
    interruption to restored service.
  - **RPO (Recovery Point Objective):** the maximum acceptable age of recovered
    data, and therefore the maximum tolerated data loss.
  - Budget the whole recovery timeline: detection, notification, escalation,
    decision and declaration, traffic movement, recovery, validation, and
    customer communication. A one-hour RTO cannot be met by a one-hour restore
    operation.
- Select the least complex and least costly strategy that meets the approved
  RTO, RPO, risk, and compliance requirements. If recovery cost exceeds the
  expected impact, the decision needs a documented business or regulatory
  reason.

## Establish the recovery foundation

1. Build for availability first. Distribute production workloads across fault
   domains, remove single points of failure, and test zonal impairment. A
   multi-AZ architecture can mitigate many local physical failures without
   requiring a cross-Region failover.
2. Treat AWS resiliency as a shared responsibility. AWS operates the underlying
   cloud infrastructure; the customer remains responsible for workload design,
   data protection, configuration, identity, quotas, capacity, and recovery
   procedures.
3. Define the recovery boundary. Inventory application components, data stores,
   queues, object storage, secrets, certificates, DNS, identity, external
   dependencies, IaC, images, and operator access. Map dependency order and
   identify dependencies that cannot run in the recovery location.
4. Make recovery reproducible. Store infrastructure, configuration, policy, and
   application deployment definitions in reviewed version control. Use IaC and
   automated deployment pipelines to create equivalent environments; back up or
   reproduce artifacts such as machine images, container images, packages, and
   configuration.
5. Protect data in layers. Use the data service's backup and point-in-time
   recovery features where appropriate; make backups immutable or protected
   against deletion; encrypt them; restrict restore permissions; and keep copies
   in a separate Region and, when justified, a separate account. Test restoring
   application-consistent data rather than only the completion of a backup job.
   For object data, enable versioning and decide how delete markers replicate;
   replicate or rebuild machine images and deployment artifacts as deliberately
   as application data. Validate the service-specific restore path, including
   promotion, permissions, encryption keys, and application consistency.
6. Size and govern the recovery environment. Confirm quotas, service limits,
   IP address capacity, encryption keys, IAM roles, DNS ownership, and network
   connectivity can support failover load. Review these prerequisites whenever
   production capacity or architecture changes. For compromise scenarios, use a
   separately governed recovery account per Region when the required isolation
   exceeds what a separate Region alone provides.

## Choose a recovery strategy

The options form a continuum. Cost and operational complexity generally rise as
RTO and RPO targets become more aggressive; the actual objectives must be
measured in exercises, not inferred from a pattern name.

| Strategy | Recovery location before an event | Typical trade-off | Key controls |
|---|---|---|---|
| Backup and restore | Backups and deployment definitions are retained; infrastructure is rebuilt or restored during recovery. | Lowest steady-state cost, but longest and most variable RTO. | Point-in-time backups, cross-Region/account copies, IaC, artifact availability, restore automation and tests. |
| Pilot light | Data replication and core services are present; non-core compute is not deployed or is switched off until recovery. | Lower running cost than a fully active environment; recovery depends on deployment and scale-out. | Replication plus backups, automated scale/deploy steps, pre-approved capacity and traffic cutover. |
| Warm standby | A scaled-down but fully functional environment is already running in the recovery Region. | Faster recovery, with ongoing cost; it must scale safely to production demand. | Continuous validation, sufficient initial capacity, quotas, data replication, tested scale-out and routing. |
| Multi-site active/active | Multiple Regions serve production traffic. | Lowest interruption for Regional loss, but highest data-consistency, security, and operating complexity. | Regional traffic policy, full-capacity tests, write-conflict design, backup/point-in-time recovery, regional-loss exercises. |

Pilot light cannot serve requests until non-core resources have been deployed
or turned on and the environment has scaled out. Warm standby can accept traffic
immediately at reduced capacity and needs only scale-out. A hot standby keeps
enough capacity for the full failover load, trading steady-state cost for less
dependence on Auto Scaling and other control-plane actions during recovery.

For server-hosted applications and databases, evaluate AWS Elastic Disaster
Recovery (DRS) where block-level replication and a pilot-light recovery
environment fit the RTO/RPO. DRS is applicable to workloads hosted on premises,
in another cloud, or on Amazon EC2; it does not replace service-native recovery
for managed services such as Amazon RDS.

For active/passive strategies, use a deliberate failover decision and automate
the approved procedure. Health-check-driven failover is appropriate only when
the signals represent user impact and false failover has been analyzed. Prefer
highly available data-plane traffic controls for the failover action where the
chosen AWS service supports them; do not assume a control-plane update will be
available during the event. For example, Amazon Route 53 Application Recovery
Controller (ARC) routing controls can provide an explicit data-plane switch;
changing Route 53 weighted-routing records or Global Accelerator traffic dials
is a control-plane operation and should not be the sole recovery dependency.

For active/active data, define the write model explicitly. A workload can route
writes to one Region, accept writes locally with a documented conflict model,
or partition write ownership by key. None of these removes the need for
point-in-time recovery from deletion, corruption, or malicious modification.

## Operate and test the plan

- Detect customer-impacting failure early. Use deep health checks that exercise
  critical journeys and combine multiple signals; a shallow process or port
  check is not sufficient. Route alerts to named owners with dashboards,
  escalation paths, communications templates, and a declaration authority.
- Keep a short, executable recovery runbook. It should identify the incident
  commander, decision authority, access path, prerequisites, ordered recovery
  actions, traffic controls, validation criteria, rollback or failback process,
  and customer or regulator communications.
- Exercise the exact path used in an emergency. Test backup restoration,
  application correctness, Regional traffic movement, scale-out, permissions,
  dependency behavior, and return to normal. Record measured RTO and RPO,
  evidence, defects, owners, and due dates.
- Run safe, regular game days. Start with isolated restores and staged failover;
  use production failure exercises only when the blast radius, observation, and
  rollback controls are approved. Recovery paths that are rarely exercised tend
  to fail when needed.
- Detect and correct drift in the recovery Region. Compare IaC, deployed
  configuration, images, policies, secrets and certificates, data protection,
  service quotas, and network/DNS settings. AWS Config and IaC drift detection
  can provide evidence, but ownership and remediation are still required.
- Review strategy after material changes: dependency additions, data-model
  changes, throughput growth, Region/account changes, incidents, failed tests,
  or revised business objectives.

## Recovery-readiness checklist

- [ ] Every workload has a business owner, approved RTO/RPO, tier, and recorded
  risk decision.
- [ ] The BCP and DR runbook cover people, dependencies, communications, and
  the complete technical recovery boundary.
- [ ] Data has tested point-in-time recovery; replication is not the only
  protection against destructive changes.
- [ ] Recovery infrastructure, application versions, configuration, identity,
  artifacts, and network controls can be recreated from versioned automation.
- [ ] Recovery account/Region quotas and capacity have been validated against
  the failover scenario.
- [ ] Detection, escalation, declaration, and traffic control are included in
  the measured RTO.
- [ ] A recent exercise proved restoration, customer-critical behavior, RTO,
  RPO, evidence capture, and follow-up ownership.
- [ ] Drift and overdue DR findings are visible to accountable owners.

## References

- AWS Well-Architected Framework, [Disaster Recovery of Workloads on AWS: Recovery in the Cloud](../../../refs/disaster-recovery-workloads-on-aws.pdf) (local source PDF)
- AWS Well-Architected Framework, [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- AWS Well-Architected Framework, [Plan for disaster recovery](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_plan-for-disaster-recovery.html)
