# Problem and Success Criteria

## Customer problem

During an incident, an operator often has to correlate an alert with Kubernetes
state, rollout history, metrics, logs, traces, and a runbook. The information
exists, but it is split across tools and interpreted differently by each person.
This creates three recurring problems:

- time is lost moving between dashboards and command-line tools;
- alert fatigue encourages shallow triage or premature conclusions; and
- diagnostic quality depends too heavily on who is on call.

The goal is not to replace the incident commander. It is to turn a raw alert
into a bounded investigation workspace: collect relevant evidence, show how the
model used it, draft an RCA, and keep every write action behind an explicit
operator decision.

## Users and workflow

| User | Need | Platform responsibility |
|---|---|---|
| On-call engineer | Understand impact and likely cause quickly | Normalize the alert, collect evidence, and present an auditable analysis |
| Service owner | Know what changed and what prevents recurrence | Connect rollout, workload, metric, log, and runbook evidence |
| Platform engineer | Keep investigation safe across namespaces | Constrain tools, credentials, target scope, retention, and write permissions |
| Incident commander | Decide whether to act | Keep analysis and execution separate and record the decision |
| Engineering manager | Improve the response system | Review outcomes, false alarms, failure modes, and regression evidence |

The implemented operator path is:

1. Alertmanager submits a firing or resolved alert.
2. The API filters by configured source namespace and upserts a deduplicated
   Issue using the alert fingerprint.
3. The operator reviews priority, severity, and source metadata.
4. The operator starts **Re-analyze**. This is deliberate: current webhook
   intake does not automatically invoke the model.
5. The service fetches bounded Prometheus and Loki context, then starts the
   Amazon Bedrock tool-use loop.
6. The model calls read-only Kubernetes and Prometheus tools and produces a
   root cause, workaround, long-term fix, and optional remediation suggestion.
7. The platform stores the transcript and tool results for review.
8. If an executable proposal is created through the attached chat tool or the
   CRD-driven Incident path, it requires a separate operator approval and passes
   through a narrow parser and namespace allowlist before execution.

## Responsibility boundaries

| Artifact | Question answered | Owner |
|---|---|---|
| RCA | What most likely happened, based on this incident's evidence? | Agent drafts; operator validates |
| Runbook | What investigation and remediation patterns are pre-approved for this failure class? | Service and platform teams |
| Playbook | Which lifecycle actions should run for a priority/severity combination? | Incident-response owner |
| Incident policy | Which alert-specific exceptions override the default playbook? | Platform governance owner |
| Remediation proposal | What single bounded action is being requested? | Agent proposes; operator approves |
| Postmortem | What should the organization learn and change? | Human owner, assisted by generated draft |

## Requirements

### Functional

- Accept Alertmanager webhook payloads without disrupting existing receivers.
- Ignore non-firing alerts for new analysis and mark matching Issues resolved
  when a resolved alert arrives.
- Filter issue creation to configured source namespaces.
- Deduplicate repeat alerts by fingerprint-derived key.
- Preserve prompt, model responses, tool inputs, tool outputs, stop reason,
  round count, and terminal result.
- Support operator start, stop, false-alarm, no-action, unresolved, and approval
  decisions.
- Never expose Kubernetes Secret values through the model tool catalog.
- Keep remediation disabled by default and grant write RBAC only to explicitly
  selected namespaces.

### Non-functional

- Return webhook acknowledgements independently of model latency.
- Bound each model response to 4,096 output tokens and the investigation to a
  configurable number of tool rounds, defaulting to 10.
- Apply 30-second HTTP timeouts to observability and Kubernetes log/tool calls.
- Limit concurrent analyses per namespace, defaulting to five.
- Retain operational artifacts on persistent volumes with configurable pruning.
- Expose health, request, model, tool, storage, issue, and decision metrics.
- Run as a non-root user with privilege escalation disabled.

## Data and model constraints

The investigation can include labels, ConfigMap values, pod logs, event
messages, resource names, and model transcripts. These can contain sensitive or
tenant-specific data even when credentials are excluded.

The current implementation provides bounded collection and persistence, but it
does not yet provide a general log-redaction pipeline, per-tenant encryption
keys, or a verified namespace check on every model-selected tool call. Those are
release gates for broader multi-tenant use.

Amazon Bedrock is called through the AWS SDK default credential chain. The Helm
chart can consume credentials from a Secret, but workload identity and
short-lived credentials are the preferred production design.

## Success criteria

Targets below define acceptance; they are not all achieved results.

| Criterion | Target | Current evidence |
|---|---:|---|
| Webhook acknowledgement p95 | < 2 seconds | Met in stored local burst runs through 500 alerts |
| Webhook request error rate | < 1% | Not recorded in the available historical summaries |
| RCA delivery | 99% within 5 minutes | Instrumented, not established over a representative window |
| RCA completion p95 | < 60 seconds | Instrumented, no complete current result set |
| Unsupported root-cause claims | 0 in release scenario suite | Rubric defined; systematic model evaluation pending |
| Required evidence present | 100% for each deterministic scenario | Scenario expectations defined; live model assertions pending |
| Tool-selection accuracy | >= 90% on reviewed scenarios | Evaluation harness pending |
| Unsafe automatic mutation | 0 | Mutation disabled by default; approval and parser controls are unit tested |
| Operator decision attribution | 100% of approvals | Login identity is written to proposal status |
| Analysis cancellation | Operator can stop a running analysis | Implemented and covered by backend tests |
| Operator acceptance | >= 80% useful or very useful | Pilot survey not yet run |

## Measurement contract

Any future outcome claim must include:

| Field | Required detail |
|---|---|
| Metric | Exact definition, including start and stop timestamps |
| Baseline | Manual or previous-system comparison |
| Result | Median and p95, not only the best run |
| Sample/window | Incident count, scenario mix, environment, and dates |
| Evidence | Exported metrics, review record, or reproducible test artifact |
| Exclusions | Failed, cancelled, duplicate, or synthetic cases and why |

The often-proposed “70% reduction in mean time to diagnose” is intentionally not
claimed here. The reference implementation has no defensible paired baseline,
sample, and completed live-RCA data set for that statement.

## Acceptance gates

A pilot is ready when all of the following are true:

- deterministic failure scenarios are run end to end against the selected
  Bedrock model;
- each conclusion is scored against expected evidence and root cause;
- tool calls are restricted to the incident's authorized namespaces;
- sensitive context is redacted before model submission and long-term storage;
- Bedrock throttling, observability failure, and storage failure runs reach a
  recorded terminal state;
- token use and estimated cost are observable; and
- operators can reject a proposal without affecting incident resolution.
