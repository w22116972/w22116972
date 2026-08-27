# Resume STAR Interview Drill

## Resume claim

> Reduced mean time to diagnose by 70% by implementing a cloud-native AIOps
> platform on Amazon EKS using Go and Amazon Bedrock, enabling agentic
> root-cause analysis through 20+ live diagnostic tools and reuse of historical
> incident RCAs to improve subsequent investigations.

Use the questions below as independent interview answers. Each architecture
decision has its own Situation, Task, Action, and Result so an interviewer can
drill into one part of the system without requiring the complete project story.

## Evidence boundary

| Resume element | Current evidence | Interview-safe position |
|---|---|---|
| Go platform for Amazon EKS | Go backend, Kubernetes controllers, Helm chart, manifests, RBAC, metrics, and storage configuration | Implemented in source; current EKS runtime was not reverified for this page |
| Amazon Bedrock agentic RCA | Bedrock provider, bounded tool loop, prompt builder, operator-started analysis, and AI trace | Implemented in source; representative live Bedrock outcomes remain unverified |
| 20+ live diagnostic tools | 17 read-only RCA tools; chat adds two callable tools, one of which creates an approval-gated proposal | Say 17 diagnostic tools or restore evidence for the 20+ scope |
| Reuse of historical RCAs | Stored RCAs, postmortems, runbooks, and feedback artifacts exist; semantic similarity retrieval is roadmap | Describe implemented learning artifacts; do not claim RAG retrieval is live |
| 70% lower MTTD | Resume statement only; no paired incident dataset is retained in this checkout | State only with the original measurement evidence and calculation |

## 1. What problem did you solve, and what did you personally own?

### Situation

During Kubernetes incidents, on-call engineers had to correlate Alertmanager,
workload state, rollout history, Prometheus, Loki, traces, and runbooks by hand.
The process depended on individual experience, slowed the first defensible
diagnosis, and left an incomplete record of why a hypothesis was accepted.

### Task

I needed to build an incident-diagnosis workflow that reduced repetitive
evidence collection while keeping the operator responsible for conclusions and
production actions. My scope covered the Go investigation path, the tool and
authority boundaries, structured RCA output, audit evidence, and deployment on
a Kubernetes platform designed for Amazon EKS.

### Action

I connected authenticated Alertmanager intake to deduplicated Issues, built an
operator-started Amazon Bedrock tool loop, exposed a fixed read-only diagnostic
catalog, parsed the output into stable RCA fields, stored every model and tool
round, and separated suggested remediation from approved typed execution.

### Result

The implementation delivers a traceable incident-to-RCA workflow rather than a
generic chatbot. The current source proves the implementation mechanics; live
operator adoption and the resume's diagnosis-time percentage require separate
outcome evidence.

## 2. Why does an alert create a pending Issue instead of automatically running the model?

### Situation

Alert storms, duplicate notifications, and noisy alerts can trigger many model
calls at once. Treating every webhook as permission to invoke an LLM would
increase cost, create uncontrolled concurrency, and let an external event cross
an important authority boundary.

### Task

I had to preserve fast, reliable alert intake while ensuring that model spend
and investigation scope remained intentional and cancellable.

### Action

The webhook authenticates the request, accepts at most 100 alerts, filters the
configured source namespaces, handles firing and resolved states, and upserts a
deduplicated pending Issue. An authenticated operator starts, stops, or repeats
analysis through a separate endpoint. Re-analysis updates the same Issue rather
than creating a disconnected investigation.

### Result

Alertmanager remains decoupled from model latency and failure. Operators gain a
clear control point for cost and investigation priority, with the tradeoff of
one additional action before RCA generation.

## 3. Why did you use an Amazon Bedrock tool loop instead of one-shot prompting?

### Situation

A one-shot prompt can summarize supplied text but cannot know which missing
piece of live cluster evidence matters next. Supplying all logs, metrics, and
Kubernetes objects up front would also increase token use and noise.

### Task

I needed the model to investigate iteratively while keeping its capabilities,
round count, and evidence sources bounded and auditable.

### Action

I implemented a Go agentic loop using the Amazon Bedrock provider. The prompt
requires evidence gathering, the model chooses from a fixed tool catalog, and
each result informs the next call. The loop defaults to a maximum of ten rounds
and stores the prompt, tool arguments, tool results, stop reason, and response.

### Result

The system can follow evidence-dependent diagnostic branches without granting
open-ended cluster or shell access. The tradeoff is additional latency, token
cost, and a larger evaluation surface than one-shot generation.

## 4. How did you design the diagnostic tools, and is the resume's 20+ count accurate?

### Situation

The model needed enough coverage to investigate common workload, rollout,
scheduling, networking, configuration, and capacity failures. At the same time,
an inflated or ambiguous tool count would be difficult to defend in an
interview.

### Task

I had to expose useful live evidence through narrow schemas, prevent sensitive
data disclosure, and keep the public tool-count claim reproducible from source.

### Action

The RCA catalog exposes 17 read-only tools for pods, logs, deployments, nodes,
events, Services, endpoints, HPA state, resource quotas, ConfigMaps, Secret key
names, StatefulSets, DaemonSets, Prometheus queries, and runbook hints. Secret
values are never returned. Chat adds `read_rca_report` and
`propose_remediation`, bringing that flow to 19 callable tools, but the proposal
tool is not a diagnostic read tool.

### Result

The current source supports a defensible claim of 17 live diagnostic tools, not
20+. I would use 17 in an interview unless the missing diagnostic integrations
and their runtime evidence can be identified and restored.

## 5. How did you prevent context size and irrelevant telemetry from overwhelming the model?

### Situation

Incident telemetry can include large log streams, high-cardinality metrics,
and unrelated namespace activity. Sending all available data increases cost and
can hide the signal the model needs.

### Task

I needed to provide enough initial context for investigation while allowing the
model to request more specific evidence only when necessary.

### Action

The backend pre-fetches a bounded incident window from Prometheus and recent
deduplicated Loki error lines, then constructs a structured prompt containing
the alert identity, workload scope, priority, severity, labels, and available
runbook guidance. Additional evidence is fetched through narrow tool calls
rather than embedded wholesale in the initial prompt.

### Result

The model starts with relevant incident context and can progressively inspect
live state. Token and cost telemetry are still incomplete, so efficiency is an
architectural property to measure rather than a verified savings claim.

## 6. How are incidents classified and routed to the correct runbook or action?

### Situation

Alert severity alone does not express both business urgency and technical
impact. Hard-coding routing logic would also require a deployment for every
operational policy change.

### Task

I needed a consistent vocabulary that connected incoming symptoms, priority,
severity, investigation guidance, and lifecycle actions while remaining
editable by operators.

### Action

The platform separates priority from severity, matches alert signals to a
runtime-editable TriageCase catalog, and uses `failure_mode` as the stable key
that connects a case, its runbook, and resulting RCA reports. IncidentPolicy
rules can override the default priority-by-severity playbook behavior without a
code redeploy.

### Result

Classification, investigation guidance, and lifecycle routing share an
explicit contract. Operators can change policy without rebuilding the service,
while deterministic matching and audit records keep the decision explainable.

## 7. How did you make an LLM-generated RCA explainable and reviewable?

### Situation

A fluent root-cause paragraph is unsafe if an operator cannot determine which
live observations support it or whether the model skipped contradictory data.

### Task

I had to make the reasoning path inspectable and return output in a form that
could be reviewed, stored, compared, and acted upon consistently.

### Action

The backend stores every model round and tool result in the AI Trace. It parses
the final response into root cause, short-term workaround, long-term fix, and
proposed remediation. The Issue retains analysis state, evidence summary,
confidence metadata, operator decisions, and links to the full RCA artifact.

### Result

Operators can inspect how the model reached its answer instead of trusting an
opaque summary. This enables later evaluation of evidence coverage and tool
selection, although an automated claim-to-evidence scorer remains roadmap.

## 8. Why is remediation separated from diagnosis?

### Situation

An LLM can produce syntactically plausible but unsafe commands, choose the wrong
namespace, or act on incomplete evidence. A human click alone does not validate
the command's full semantics.

### Task

I needed to preserve the speed of AI-assisted diagnosis without allowing model
text to become direct production authority.

### Action

The RCA catalog is read only. A remediation suggestion must cross a separate
authenticated proposal path and receive explicit operator approval. The
controller then validates the allowed action, resource type, target namespace,
replica bounds, and rollback information before using typed Kubernetes APIs.
The Helm chart disables write-capable remediation by default.

### Result

The design creates three independent controls: diagnosis cannot write,
approval does not bypass validation, and execution is limited to a small action
library. The narrower automation surface is an intentional tradeoff for blast
radius and auditability.

## 9. How does the system reuse historical incidents to improve later investigations?

### Situation

Incident knowledge is often lost in chat threads or isolated postmortems, so
teams repeatedly rediscover the same failure pattern.

### Task

I needed to preserve reviewed investigation artifacts and create a feedback
path into future operational guidance without feeding unreviewed model output
back into the system automatically.

### Action

The platform stores RCA reports, operator decisions, remediation execution
records, postmortems, and runbooks. Postmortem action items can identify missing
or weak automation, and engineers can improve the runbook library connected by
`failure_mode`. The current implementation uses this human-reviewed runbook
feedback path; semantic similarity search over historical RCAs is a planned RAG
phase, not a live feature.

### Result

Reviewed incident knowledge can improve later runbook-guided investigations.
The resume phrase "reuse of historical incident RCAs" should not be described
as automatic semantic retrieval until the index, authorization filters,
evaluation, and source citations are implemented and tested.

## 10. How did you calculate the 70% reduction in mean time to diagnose?

### Situation

Webhook latency, time to generate an RCA, time to acknowledge, time to diagnose,
and time to resolve measure different parts of incident response. Mixing them
can create a percentage that sounds strong but is not reproducible.

### Task

I needed an exact start and stop definition and a comparable baseline that
isolated diagnosis time from remediation time and unrelated operational change.

### Action

For each incident, diagnosis starts when the firing alert is accepted and ends
when an operator records a reviewed root-cause hypothesis with the required
evidence. The primary resume calculation is:

```text
diagnosis_time = reviewed_diagnosis_at - alert_accepted_at

mean_reduction_percent =
  (baseline_mean - assisted_mean) / baseline_mean * 100
```

I would pair comparable incident classes, document exclusions and confounders,
retain raw timestamps and reviewer decisions, and report sample size, median,
p90, and p95 alongside the arithmetic mean.

### Result

The resume reports a 70% reduction, but the current checkout does not retain the
paired incident dataset needed to reproduce it. I would state 70% only with the
original baseline, assisted window, sample, formula, and review evidence;
webhook acknowledgement measurements are not a substitute.

## 11. How does the design behave during an alert storm or dependency failure?

### Situation

An alert storm can overload model concurrency, while unavailable Prometheus,
Loki, Amazon Bedrock, Kubernetes APIs, or storage can leave an investigation
partial or misleading.

### Task

I needed intake to remain bounded and failures to be visible without presenting
an incomplete analysis as a successful RCA.

### Action

The webhook limits request size and deduplicates Issues before analysis. The
analysis path applies namespace concurrency control and a maximum tool-round
count, records failure or cancellation state on the Issue, and preserves the
analysis log. If initial observability context fails, the system can continue
with an empty bounded context and explicit live tool calls rather than silently
inventing telemetry.

### Result

The service has bounded intake and investigation controls plus visible terminal
states. Durable queues, retry and replay, highly available workers, dependency
failure testing, and end-to-end drain metrics remain release work before a
production-scale reliability claim.

## 12. What are the most important security and multi-tenancy risks?

### Situation

The model processes operational data and can choose tool arguments. In a
multi-tenant cluster, a model-provided namespace or broadly scoped ServiceAccount
could expose another workload's metadata even if every tool is read only.

### Task

I needed to protect Secret values and write operations immediately while
identifying the remaining authorization gaps that block a production
multi-tenant claim.

### Action

The tool catalog returns only Secret key names, keeps diagnosis read only, and
places execution behind authentication, approval, deterministic validation,
and an allowlisted action library. I identified two release gates: enforce the
configured source namespace at every tool call and replace broad cluster-read
RBAC with narrower tenant-aware permissions.

### Result

The current design limits sensitive values and mutation risk, but it is still a
pilot rather than a proven multi-tenant production operator. Naming those gaps
defines concrete security work instead of hiding it behind read-only language.

## 13. Why build this platform instead of buying a complete AIOps product?

### Situation

Managed or third-party AIOps products can accelerate feature delivery, but they
may not fit in-cluster RBAC, private connectivity, data residency, offline
deployment, audit-trace, or deterministic remediation requirements.

### Task

I needed to decide which responsibilities should remain inside the platform and
which managed capabilities could be adopted without surrendering the security
and control model.

### Action

I kept alert normalization, Kubernetes authorization, tool execution, evidence
storage, approval, and remediation validation in the Go platform. Amazon
Bedrock supplies model inference behind an explicit provider boundary. Managed
memory, retrieval, or orchestration can be evaluated separately behind feature
flags and must preserve non-AWS or offline fallback paths where required.

### Result

The architecture can use managed inference without outsourcing operational
authority. It costs more engineering effort than adopting a complete product,
but preserves auditability, deployment flexibility, and deterministic safety
controls. Cost or accuracy improvements from future managed components remain
targets until measured.

## 14. What would you change before production adoption?

### Situation

The pilot demonstrates the source architecture and deterministic scenarios but
does not yet prove representative live model quality, multi-tenant isolation,
high availability, or diagnosis-time improvement.

### Task

I needed to define release gates that convert a convincing demo into an
operationally supportable service.

### Action

I would add tool-level namespace authorization, narrower RBAC, sensitive-data
redaction, durable queue and replay, highly available persistence and workers,
token and cost telemetry, Tempo and service connectors, dependency chaos tests,
and a representative Bedrock evaluation set. Historical-RCA retrieval would
also require tenant filters, freshness policy, citations, and measured
relevance before activation.

### Result

These gates provide a staged adoption path: shadow mode, reviewed RCA quality,
selected operator use, and only then narrowly approved remediation. The current
portfolio therefore presents the system as a pilot with explicit next steps,
not an autonomous production operator.

## Related implementation phases

1. [Problem and success criteria](1-problem-and-success-criteria.md)
2. [Agentic architecture and guardrails](2-agentic-architecture-and-guardrails.md)
3. [Platform implementation](3-platform-implementation.md)
4. [Evaluation and results](4-evaluation-and-results.md)
5. [Operations and adoption](5-operations-and-adoption.md)
