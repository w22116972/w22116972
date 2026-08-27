# Customer Expansion and Modernization Roadmap

> **Phase:** 5 Operate · **Category:** Modernization
> **Priority topic:** 33
> **Evidence standard:** Engagement roadmap template. It does not claim customer
> expansion, revenue, adoption, or an approved future program.

## Abstract

Expansion begins after the first migration proves value and the customer can
operate it. The roadmap is driven by business outcomes and capability gaps, not
by a list of AWS services to consume.

## Roadmap horizons

| Horizon | Objective | Candidate outcomes |
|---|---|---|
| Stabilize | Close production and ownership gaps | SLOs, security findings, cost attribution, recovery and independent operations |
| Standardize | Reuse what the first migration proved | Golden paths, GitOps adoption, identity, observability and migration-wave templates |
| Optimize | Improve unit economics and resilience | Right-sizing, Karpenter, Spot, Graviton, data lifecycle, DR and performance work |
| Modernize | Change application and data architecture selectively | Domain extraction, event-driven services, serverless, database modernization and API evolution |
| Innovate | Apply new capabilities to a validated business problem | AI/ML, Agentic AIOps, analytics, edge or industry-specific solutions with governance |

## Prioritization model

Score each candidate by customer outcome, urgency, risk reduction, dependency,
readiness, evidence quality, delivery effort, operating cost, reversibility, and
ability to reuse the capability across workloads. A high technology-interest
score cannot substitute for an accountable business owner and success metric.

## Engagement sequence

1. Review the original success criteria, unresolved risks, incidents, adoption,
   unit cost, and operator feedback.
2. Facilitate an outcome workshop with business, engineering, security,
   operations, finance, and data owners.
3. Build option papers with current state, target outcome, alternatives,
   dependencies, estimated effort, risk, evidence plan, and exit strategy.
4. Select one or two bounded proofs of value with production-like validation.
5. Convert successful proofs into funded waves with owners, milestones, adoption
   measures, handoff, and retirement of superseded mechanisms.
6. Stop or redesign candidates that do not meet the agreed outcome.

## Roadmap evidence

- customer-approved outcome and accountable owner;
- baseline, target, measurement formula, and evidence source;
- architecture decision with rejected alternatives and tradeoffs;
- security, compliance, resilience, cost, and operating review;
- proof-of-value results and production-adoption gate;
- roadmap status separated into proposed, approved, implemented, validated, and
  retired states.

## Portfolio boundary

This roadmap demonstrates consultative architecture and customer expansion
thinking. Only completed, validated horizons should be presented as delivered
outcomes; proposals remain explicitly labeled as roadmap.

## References

- [Customer Delivery and Handoff](../../practices/aws/customer-delivery-handoff.md)
- [Operations and Handoff](5-operations-and-handoff.md)
