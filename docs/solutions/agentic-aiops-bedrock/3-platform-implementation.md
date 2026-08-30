# Platform Implementation

## Abstract

This phase records the reference implementation as inspected and tested on a
stated date, covering the alert and issue lifecycle, context assembly, API
surface, Kubernetes resource model, remediation, and storage. It separates
executable behavior from configuration, examples, and roadmap material,
because those had diverged in the original project documentation.

## Source-verified snapshot

This document describes the reference implementation as inspected and tested on
2026-08-19. It separates executable behavior from configuration, examples, and
roadmap material because those had diverged in parts of the original project
documentation.

## Components

| Component | Technology | Responsibility |
|---|---|---|
| Backend | Go, controller-runtime, HTTP API | Issue intake, model orchestration, tools, CRDs, metrics, persistence, and reconcilers |
| Model provider | AWS SDK for Go, Bedrock Converse API | Tool-calling completion loop |
| Operator interface | React and Vite, served by nginx | Issues, analysis controls, reports, playbooks, runbooks, postmortems, settings, and AI Trace |
| Observability adapters | Prometheus and Loki HTTP APIs | Pre-fetch metrics and error logs |
| Kubernetes tools | controller-runtime client and API calls | Read workload, rollout, scheduling, networking, configuration, and event evidence |
| Persistence | JSON and Markdown on persistent volumes | Issues, settings, traces, RCAs, postmortems, executions, and predictions |
| Packaging | Helm | Deployments, Services, ingress, CRDs, RBAC, configuration, storage, alerts, and network policy |
| Delivery | CI pipeline | Frontend tests, image builds, registry push, Helm upgrade, and rollout wait |

One backend binary runs the HTTP API and controller-runtime manager. The
frontend is a separate nginx Deployment and does not share the backend process
or filesystem.

## Alert and issue lifecycle

The current webhook contract is deliberately read-first:

```text
POST /webhook
  -> validate JSON and cap request at 100 alerts
  -> resolved: mark matching Issue resolved
  -> firing: check configured source namespace
  -> normalize priority, severity, labels, and source
  -> upsert JSON Issue by dedupe key
  -> broadcast Issue state over SSE
  -> resolve playbook/policy actions
```

The response contains `queued` and `dropped`, but current webhook intake keeps
both at zero because it does not enqueue automatic analysis. Backend tests
assert that even a critical firing alert remains pending until an operator
starts analysis.

Playbook and IncidentPolicy selection is implemented. The runtime dispatcher is
currently created without action handlers, so actions such as `generate_rca`,
`create_jira`, `notify_slack`, and `generate_postmortem` are recorded as skipped
or abstract rather than executed from this path.

## Operator-started RCA

`POST /issues/{id}/reanalyze` changes the Issue to `analysis_running`,
broadcasts the change, and starts the analysis asynchronously.

The worker then:

1. acquires the per-namespace concurrency slot;
2. fetches Prometheus and Loki context;
3. builds the structured RCA prompt;
4. calls Amazon Bedrock through the provider interface;
5. executes model-selected tools and feeds their results back;
6. persists the entire transcript as an analysis log;
7. parses the final response into RCA fields and a non-executable remediation
   suggestion; and
8. updates and broadcasts the Issue.

An operator can cancel a running analysis. Provider errors and rate-limit
rejections return the Issue to an actionable failed state instead of presenting
an incomplete answer as success.

### Output and trace

The stored trace contains:

- system and user prompts;
- each assistant response and stop reason;
- tool name and JSON arguments;
- tool output and error state;
- round count and timestamps; and
- final model text or provider error.

The AI Trace view exposes this history and groups the final RCA fields. This is
the main audit mechanism for checking whether a conclusion is supported.

## Context implementation

### Prometheus

The pre-fetcher runs four namespace-scoped 30-minute queries:

- CPU throttling;
- memory working set;
- container restart count; and
- HTTP 5xx rate.

Only the latest available sample for each query enters the initial prompt. The
model can also call a PromQL instant-query tool during investigation.

### Loki

The pre-fetcher queries recent lines containing `ERROR`, `WARN`, or `FATAL`,
limits the response to 200 lines, removes duplicate lines, and adds at most 50
lines to the prompt.

### Tempo

The Tempo endpoint is configurable and included in observability health
reporting, but the inspected context fetcher does not issue a trace query.
Trace-based diagnosis is therefore a stub, not a live capability.

### Runbook guidance

Runbook storage, matching, editing, and APIs are implemented. The RCA tool named
`load_runbook` currently returns only a generic acknowledgement; it does not
return the matched runbook's structured investigation steps. Full runbook
injection remains incomplete in this execution path.

## API surface

The principal routes are:

| Route | Purpose | Protection |
|---|---|---|
| `POST /webhook` | Alertmanager intake | Webhook token when configured |
| `GET /health`, `/healthz`, `/metrics` | Probes and Prometheus scraping | Unauthenticated by design |
| `GET /issues` | Issue list | Signed operator session |
| `POST /issues/{id}/reanalyze` | Start RCA | Signed operator session |
| `POST /issues/{id}/analysis/stop` | Cancel RCA | Signed operator session |
| `GET /issues/{id}/analysis-log` | Full evidence trace | Signed operator session |
| `POST /issues/{id}/decisions` | Resolve, reject, or approve | Signed operator session |
| `GET /reports`, `/postmortems`, `/runbooks` | Operational artifacts | Signed operator session |
| `GET /stream` | Issue and result events | Signed operator session |
| `GET /docs`, `/openapi.*` | API reference | Intended to be protected at ingress |

The API and frontend use an eight-hour HMAC-signed session cookie. Login adds
per-IP rate limiting, account backoff after repeated failures, and a global cap
on concurrent bcrypt checks.

## Kubernetes resource model

Six custom resources provide an API-native operational model:

| CRD | Purpose |
|---|---|
| `Incident` | Alert metadata and RCA lifecycle state |
| `RCAReport` | Structured root cause, workaround, long-term fix, and proposal |
| `RemediationProposal` | Approval and execution status for a write action |
| `PredictionRecord` | Forecast signal, confidence, and state |
| `IncidentPlaybook` | Default priority-by-severity action matrix |
| `IncidentPolicy` | Ordered exceptions with match conditions and lifecycle hooks |

The Incident reconciler supports an independent CRD-driven analysis path when
an Incident has the explicit `auto-analyze` annotation. The HTTP webhook path
does not currently create or annotate those Incident resources, so this should
not be described as automatic webhook-to-RCA behavior.

## Remediation implementation

Remediation is a separate reconciler. The attached operator chat has a
`propose_remediation` tool that creates a proposal for an Issue, and the
annotated Incident CRD path can also create one after analysis. The ordinary
Issue Re-analyze path does not create the proposal CR.

The reconciler reacts only when a
`RemediationProposal` reaches `Approved`, parses the proposed action, validates
its exact grammar and target namespace, and then calls the typed Kubernetes
client.

| Accepted form | Additional control |
|---|---|
| Restart Deployment | Deployment name and namespace validation |
| Scale Deployment | Required replica count from 0 through 50 |
| Delete Pod | Pod name and namespace validation |

No original model string reaches a shell. Chart defaults keep remediation off.
When enabled, Helm creates namespaced write Roles only for the configured
remediation namespaces.

## Storage and streaming

Issues and traces are JSON files. RCAs, postmortems, and execution records are
Markdown. The chart mounts separate persistent volumes and uses a `Recreate`
strategy for the single backend replica, consistent with ReadWriteOnce storage.

Configurable lifecycle processing deletes or archives reports, postmortems, and
predictions older than the retention window. Archiving into the same volume
preserves history but does not recover capacity.

SSE uses an in-memory broadcaster with a 32-event buffer per client and a
15-second heartbeat. Slow-consumer events are dropped instead of blocking the
service, and there is no replay cursor. The UI also polls, which provides a
state-refresh fallback.

## Helm and deployment controls

The chart includes:

- separate backend and frontend Deployments;
- liveness and readiness probes;
- CPU and memory requests and limits;
- CRDs and least-write RBAC toggles;
- optional ingress and TLS issuer configuration;
- optional API-documentation ingress protection;
- Alertmanager configuration and SLO alert rules;
- persistent volume claims and retention settings; and
- optional network policy.

The containers run as a non-root UID/GID with privilege escalation disabled.
The current backend filesystem is writable because it persists local artifacts.
The optional network policy restricts the metrics port but allows all egress;
destination-specific egress rules are a production hardening item.

## Delivery pipeline

The reference CI definition contains frontend tests, backend and frontend image
builds, registry publication, a Helm upgrade, and a rollout status gate. Two
limitations matter:

- backend Go tests are not a CI stage in the inspected pipeline; and
- the deploy stage does not run the deterministic scenario suite or evaluate a
  Bedrock response before promotion.

Recommended gates are backend tests and static analysis, Helm render policy
checks, scenario smoke tests with the mock provider, a controlled Bedrock eval,
and an explicit production promotion.

## Capability inventory

| Capability | State |
|---|---|
| Alert filtering, resolution, normalization, and deduplication | Implemented |
| Operator-started Bedrock tool loop | Implemented |
| 20 read-only Kubernetes and Prometheus diagnostic tools | Implemented |
| Prometheus and Loki context pre-fetch | Implemented |
| Full model/tool transcript and AI Trace UI | Implemented |
| Human approval and typed remediation execution | Implemented for chat/CRD-created proposals; disabled by default |
| Proposal creation directly from Issue Re-analyze | Not implemented |
| Runtime playbook and policy editing | Implemented |
| Automatic webhook-to-RCA orchestration | Not wired in current HTTP path |
| Tempo trace fetch | Stub |
| Database and model-gateway diagnostic connectors | Stub |
| Jira creation and playbook Slack action | Abstract/unwired in dispatcher |
| Durable work queue, retry, and dead-letter handling | Planned |
| Model token/cost telemetry | Planned |
| General sensitive-data redaction | Planned |
| Strong per-tool namespace isolation | Planned |
| Multi-replica/high-availability persistence | Planned |
