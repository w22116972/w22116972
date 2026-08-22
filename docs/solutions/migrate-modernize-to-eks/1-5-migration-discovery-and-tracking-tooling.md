# Migration Discovery and Tracking Tooling

## Role in the journey

**Category:** Migration
**Priority topic:** 29
**Evidence status:** Recommended enterprise migration pattern; no customer-scale
tool deployment or completed portfolio assessment is claimed.

Tooling supports discovery and governance, but it does not choose a migration
strategy. Business criticality, dependencies, data sensitivity, change windows,
and owner decisions remain authoritative.

## Discovery model

| Need | Candidate mechanism | Decision output |
|---|---|---|
| Directional portfolio inventory | Existing CMDB, billing data, interviews, and file import | Initial server, application, owner, and environment register |
| VMware or server utilization | AWS Transform discovery or AWS Application Discovery Service collectors | CPU, memory, configuration, and source-platform evidence |
| Process and connection detail | Discovery Agent or approved application telemetry | Dependency hypotheses requiring owner validation |
| Database assessment | DMS Fleet Advisor, engine-native inventory, and DBA review | Compatibility, sizing, licensing, and migration candidates |
| Program tracking | AWS Migration Hub or an approved delivery system | Wave, application, server, risk, and status view |

Agentless discovery is useful for broad VMware inventory but cannot prove every
process, connection, or business dependency inside a guest. Tool output is
therefore reconciled with application-owner interviews, traffic evidence, DNS,
database relationships, batch schedules, and incident history.

## Implementation sequence

1. Define the portfolio fields, source owners, retention, and regional data
   handling requirements before deploying a collector.
2. Select a Migration Hub home Region and confirm whether collected metadata is
   permitted there.
3. Pilot discovery on a representative, non-critical segment.
4. Reconcile duplicates, stale systems, shared databases, and unknown owners.
5. Group infrastructure into applications and dependency-bounded migration
   waves.
6. Attach a 7-R decision, target architecture, test owner, cutover window, and
   rollback path to every in-scope application.
7. Freeze a versioned baseline for business-case and wave planning; continue to
   record approved changes after the freeze.

## Acceptance evidence

- coverage and freshness are known for every discovery source;
- owners approve critical dependency maps and unresolved assumptions;
- sensitive metadata is access-controlled and retained for an approved period;
- every wave traces from portfolio record to migration strategy and exit gate;
- tracking status is reconciled with actual deployment and traffic evidence;
- tool removal, agent uninstall, and data-retention procedures are documented.

## Failure and rollback

If a collector affects source performance, exceeds its authorized scope, or
stores data in an unacceptable location, stop collection, preserve the audit
record, remove the collector through its supported procedure, and fall back to
file import plus targeted interviews. Previously collected data is not treated
as current until it is revalidated.

## References

- [AWS Application Discovery Service](https://docs.aws.amazon.com/application-discovery/latest/userguide/what-is-appdiscovery.html)
- [AWS Transform discovery tool](https://docs.aws.amazon.com/transform/latest/userguide/discovery-tool.html)
