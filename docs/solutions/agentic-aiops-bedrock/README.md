# Agentic AIOps on Amazon Bedrock and Amazon EKS

## Abstract

This case study implements a read-first, human-governed AIOps assistant for
Kubernetes incident diagnosis. Alertmanager creates a normalized, deduplicated
Issue. An operator can then start an Amazon Bedrock investigation that combines
bounded Prometheus and Loki context with 17 read-only Kubernetes, Prometheus,
and runbook tools. The platform preserves the complete prompt, tool-call,
evidence, and response trace before presenting a root cause, workaround,
long-term fix, and optional remediation suggestion.

It is more than a dashboard aggregator because it selects the next diagnostic
step from live evidence. It is more controlled than a generic chatbot because
the tool catalog is fixed, analysis is separate from mutation, and supported
write actions require an authenticated human approval plus deterministic
validation.

The implementation is a pilot, not an autonomous production operator. Current
webhook intake does not automatically run the model, Tempo and service
connectors are incomplete, and live Bedrock accuracy and time-saving outcomes
have not been established.

## Resume alignment

> Reduced mean time to diagnose by 70% by implementing a cloud-native AIOps
> platform on Amazon EKS using Go and Amazon Bedrock, enabling agentic
> root-cause analysis through 20+ live diagnostic tools and reuse of historical
> incident RCAs to improve subsequent investigations.

The numbered pages below provide the architecture and implementation depth
behind this resume statement. The
[STAR interview walkthrough](6-resume-star-interview.md) connects the complete
story and separates the resume-reported outcome from what the current source
checkout independently proves. In particular, the checked-in RCA path exposes
17 read-only diagnostic tools; chat adds two tools, and semantic retrieval over
historical RCAs remains a roadmap capability.

## Primary architecture

```mermaid
flowchart LR
    A[Alertmanager] -->|firing or resolved alert| B[Webhook intake]
    B --> C[Deduplicated Issue]
    C -->|operator starts analysis| D[Bounded context]

    subgraph O[Observability]
      P[Prometheus]
      L[Loki]
      T[Tempo - adapter pending]
    end

    P --> D
    L --> D
    T -.-> D

    D --> E[Amazon Bedrock reasoning loop]
    E <-->|approved read-only calls| F[Kubernetes and Prometheus tools]
    F --> G[Evidence and transcript]
    E --> H[Structured RCA draft]
    G --> H
    H --> I{Human decision}
    I -->|reject or no action| J[Issue record]
    I -->|request executable action through chat or CRD flow| Q[AwaitingApproval proposal]
    Q -->|authenticated approval| K[Validated remediation controller]
    K -->|typed API call in allowlisted namespace| M[Amazon EKS]
    J --> N[Postmortem and runbook feedback]
    K --> N
```

The trust model has three deliberate breaks:

1. an alert can create an Issue but does not automatically earn a model call;
2. a model can propose a command but cannot directly execute it; and
3. operator approval is necessary but not sufficient—the controller still
   validates the full action and namespace.

## Incident-to-RCA lifecycle

1. **Ingest** — accept at most 100 alerts in a webhook request, verify the
   webhook token when configured, and ignore sources outside the configured
   namespace list.
2. **Normalize** — map labels and triage data into independent priority and
   severity dimensions, then upsert the Issue by dedupe key.
3. **Govern** — let runtime IncidentPolicy rules override the default
   priority-by-severity playbook matrix.
4. **Start** — an authenticated operator begins or cancels analysis.
5. **Collect** — query a 30-minute Prometheus window and recent deduplicated
   Loki error lines.
6. **Investigate** — Bedrock selects from workload, rollout, scheduling,
   networking, configuration, event, metric, and runbook tools.
7. **Explain** — parse the final response into root cause, workaround,
   long-term fix, and proposed remediation text.
8. **Audit** — store and display every model round and tool result.
9. **Decide** — the operator resolves, rejects, reruns, or marks false alarm.
   An executable proposal can be created through the attached chat tool or the
   separately annotated Incident CRD path, then approved by the operator.
10. **Learn** — preserve the RCA, decision, execution outcome, and postmortem
    inputs for improving runbooks and future evaluation.

## Guarded action boundary

The RCA model-facing tools are read only. Kubernetes Secret values are never
returned by the tool catalog. The ordinary Re-analyze path stores remediation
text as a suggestion; it does not create an executable proposal. Proposal
creation is implemented in the attached chat and annotated Incident CRD paths.
Write-capable remediation is disabled in the chart by default and accepts only
restart Deployment, scale Deployment within a 0-50 replica bound, or delete
Pod. The controller rejects unexpected syntax, flags, resources, names, and
namespaces and uses the typed Kubernetes client rather than a shell.

Two gaps prevent a multi-tenant production claim: diagnostic tools currently
accept a model-provided namespace without enforcing the configured source list,
and the service account has broad cluster read access. Tool-level authorization
and narrower RBAC are top release gates.

## Implemented versus incomplete

| Implemented | Incomplete or roadmap |
|---|---|
| Alert intake and Issue deduplication | Automatic webhook-to-RCA worker |
| Operator-started analysis and cancellation | Durable queue, retry, and replay |
| Amazon Bedrock provider and tool loop | Token and cost telemetry |
| Prometheus and Loki context | Tempo trace retrieval |
| 17 diagnostic tools | Database and model-gateway connectors |
| Full AI Trace and operator UI | Automated claim-to-evidence scoring |
| Approval-gated typed remediation for chat/CRD proposals | Direct proposal creation from Re-analyze |
| Helm, CRDs, metrics, and storage | Highly available persistence and workers |
| Scenario fixtures and local test suite | Representative live Bedrock outcome data |

## Verification and measured evidence

Local source verification on 2026-08-19 produced:

- passing tests across all backend Go packages;
- 21 passing frontend test files with 115 tests;
- a successful production frontend build;
- passing Helm lint and template rendering; and
- valid rendering for all 11 deterministic incident overlays, including OOM,
  throttling, CrashLoopBackOff, disk full, missing endpoints, failed rollout,
  exhausted quota, HPA saturation, readiness failure, pending storage, and HTTP
  500 errors.

Historical local burst artifacts show webhook acknowledgement p95 of 0.643s for
50 alerts, 1.012s for 100 alerts, and approximately 1.000s for 500 alerts. The
artifacts do not contain completed RCA drain, error-rate, or model-quality
summaries, so the measurements must not be interpreted as end-to-end incident
performance.

No 70% reduction in diagnosis time is claimed. A valid claim requires paired
baseline incidents, a representative sample, exact start/stop definitions, and
reviewed outcomes.

## Key decisions

| Decision | Reason | Tradeoff |
|---|---|---|
| Operator-started analysis | Controls model spend and keeps humans engaged | Adds one action before evidence collection |
| Tool calling instead of one-shot prompting | Lets evidence determine the next diagnostic step | More latency, cost, and evaluation surface |
| Deterministic execution controller | Prevents arbitrary model-generated commands | Supports only a narrow action library |
| File artifacts for the pilot | Simple, inspectable, and portable | Single-replica and durability limitations |
| Explicit implemented/stub inventory | Keeps the public architecture defensible | Makes product gaps visible |

## Adoption recommendation

Start in shadow mode with remediation disabled. Score each deterministic
scenario for root-cause correctness, required evidence, tool selection,
uncertainty, usefulness, and proposal safety. Before active incident use, add
tool-level namespace enforcement, sensitive-data redaction, completed dependency
failure tests, and token/cost telemetry. Enable selected low-risk remediation
only after the analysis path has earned operator trust.

## Phases

1. [Problem and success criteria](1-problem-and-success-criteria.md)
2. [Agentic architecture and guardrails](2-agentic-architecture-and-guardrails.md)
3. [Platform implementation](3-platform-implementation.md)
4. [Evaluation and results](4-evaluation-and-results.md)
5. [Operations and adoption](5-operations-and-adoption.md)

## Appendix

- [Resume STAR interview walkthrough](6-resume-star-interview.md)
