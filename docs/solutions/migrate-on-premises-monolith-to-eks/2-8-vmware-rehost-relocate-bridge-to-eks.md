# VMware Rehost or Relocate Bridge to Amazon EKS

> **Phase:** 2 Design · **Category:** Migration
> **Priority topic:** 30
> **Evidence standard:** Optional datacenter-exit pattern. The retained case
> study does not prove a VMware migration, AWS Transform deployment, or AWS
> Application Migration Service program.

## Abstract

Not every legacy workload can be refactored before a datacenter deadline. A
temporary rehost or relocate wave can move infrastructure risk first, followed
by application modernization to EKS under a separate approval and rollback
model.

## Strategy boundary

```text
source VMware or bare-metal estate
  -> discover and group dependencies
    -> retain, retire, rehost, or relocate urgent workloads
      -> stabilize and validate on AWS
        -> assess application modernization readiness
          -> containerize or refactor selected capabilities to EKS
            -> retire temporary EC2 or VMware landing targets
```

The bridge is justified only when it reduces a real schedule, hardware, lease,
licensing, or operational risk. It must not become an indefinite double
migration with no funded modernization exit.

## Key decisions

| Path | Prefer when | Exit condition |
|---|---|---|
| Rehost to Amazon EC2 | Time is constrained and the guest can move with minimal application change | Application is stable on EC2 or approved for later modernization |
| Relocate a VMware environment | VMware-level continuity is required and the target service is approved | Workload remains intentionally on VMware or has a funded exit roadmap |
| Replatform directly to EKS | Application can be containerized without combining unacceptable data and platform risk | Image, release, traffic, dependency, and rollback contracts pass |
| Refactor after rehost | Business value justifies service or data redesign after infrastructure risk is removed | New service boundary operates independently and temporary host is retired |

## Delivery controls

1. Discover servers, applications, databases, network flows, owners, licenses,
   and maintenance windows.
2. Group complete dependency units and define rehost/relocate test and rollback
   criteria.
3. Establish landing-zone, identity, connectivity, logging, backup, and cost
   controls before replication.
4. Run a non-production or test wave, then reconcile configuration and
   application behavior—not only server boot state.
5. Cut over with DNS, data, integration, and user-journey validation.
6. Record the modernization trigger, owner, target date, and retirement cost for
   every temporary landing target.

## Claim boundary

Server replication, a successful test launch, or a running EC2 instance does
not prove application migration. The outcome requires dependency, data,
security, performance, user-path, operations, and rollback evidence.

## References

- [AWS Transform migrations including VMware](https://docs.aws.amazon.com/transform/latest/userguide/transform-app-vmware.html)
- [Migration discovery and tracking](1-5-migration-discovery-and-tracking-tooling.md)
