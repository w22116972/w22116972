# Industry-Specific Workload Architecture

> **Phase:** 3 Execute · **Category:** Modernization
> **Priority topic:** 34
> **Evidence standard:** Discovery and architecture framework only. No
> semiconductor, manufacturing, public-sector, or financial-services customer
> outcome is claimed without retained evidence.

## Abstract

Industry labels do not determine architecture. They expose constraints that
must be translated into workload-specific requirements, controls, and tests.

## Constraint map

| Workload context | Questions to answer | Candidate architectural implications |
|---|---|---|
| Semiconductor and EDA/HPC | License locality, large files, scheduler semantics, placement, throughput, burst capacity | Hybrid connectivity, shared storage, batch scheduling, capacity reservations, specialized instances |
| Manufacturing and IoT/OT | Intermittent links, protocol translation, plant safety, edge autonomy, device identity | Store-and-forward, edge processing, event ingestion, segmented networks, controlled cloud-to-OT commands |
| Public sector | Data classification, procurement, audit, residency, accessibility, continuity | Account and region restrictions, approved services, evidence retention, strong separation, documented recovery |
| Financial services | Transaction integrity, low latency, regulatory evidence, fraud controls, reconciliation | Idempotency, immutable audit, encryption, strong identity, multi-AZ recovery, controlled change windows |

## Architecture method

1. Start with the business process, consequence of failure, users, regulators,
   data classes, locations, peak periods, and external dependencies.
2. Convert each constraint into a measurable non-functional requirement.
3. Select EKS only where its scheduling, policy, ecosystem, portability, or
   operating model creates value.
4. Keep industry-specific managed services and integrations outside the cluster
   unless the application requires Kubernetes ownership.
5. Design degraded modes for lost connectivity, unavailable licenses, delayed
   events, capacity shortage, and regional service restrictions.
6. Validate with domain owners and representative workload tests.

## Required evidence

- named source for every regulatory or data-residency requirement;
- workload profile covering latency, throughput, concurrency, storage, network,
  availability, and recovery;
- accepted shared-responsibility and operational ownership model;
- allowed-service, region, encryption, identity, logging, and retention
  decisions;
- load, failure, security, reconciliation, and recovery results;
- explicit statement of which industry claims are illustrative versus observed.

## Portfolio use

This page demonstrates the ability to translate industry constraints into
architecture. It should not be presented as delivery experience in an industry
unless the corresponding customer engagement and evidence can be defended.

## References

- [Enterprise AWS Networking and Security](../../practices/aws/enterprise-networking-security.md)
- [Customer Delivery and Handoff](../../practices/aws/customer-delivery-handoff.md)
