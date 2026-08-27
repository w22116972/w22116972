# Operations and Adoption

## Abstract

This phase puts the platform into service as an investigation assistant rather
than an autonomous incident responder. It defines the operational signals,
service objectives, failure modes, back-pressure and idempotency behavior,
model and cost governance, and how incidents affecting the AIOps platform
itself are handled.

## Operating model

This platform should enter service as an investigation assistant, not an
autonomous incident responder. The safe default is:

- alerts create or update Issues;
- operators choose when to invoke the model;
- evidence and model traces remain visible;
- remediation remains disabled until a team explicitly enables scoped write
  permissions; and
- the incident commander owns the final diagnosis and action.

## Operational signals

The service exposes metrics for:

- HTTP requests and latency;
- model calls, duration, outcome, and stop reason;
- tool calls and duration;
- provider and report-write errors;
- active SSE clients;
- stored bytes and files;
- issue states and operator decisions;
- playbook action outcomes;
- false-alarm and suppression-rule activity; and
- remediation execution outcomes.

The included alert rules cover service availability, request errors and
latency, RCA failures, provider errors, storage, and selected deterministic test
scenarios. A dashboard is supplied for the main service indicators.

Token counts and estimated Bedrock cost are not currently exported.

## SLOs and error budgets

Recommended pilot objectives are:

| Objective | Target | Response when breached |
|---|---:|---|
| Webhook acknowledgement p95 | < 2 seconds | Disable optional enrichment and inspect intake saturation |
| Non-model API p95 | < 500 ms | Check storage latency, auth pressure, and file count |
| RCA completion p95 | < 60 seconds | Reduce context/round budget or use a lower-latency model |
| RCA delivery | >= 99% within 5 minutes | Stop expansion and repair queue/retry behavior |
| Unsafe execution | 0 | Disable remediation immediately and review approval/RBAC logs |
| Unsupported decisive claim | 0 in release suite | Reject prompt/model release |

Only the webhook p95 target has supporting historical measurements, and those
measure acknowledgement rather than completed analysis.

## Failure modes and runbooks

| Failure | Current behavior | Operator response | Needed improvement |
|---|---|---|---|
| Bedrock error or throttling | Analysis becomes failed and trace records provider error | Retry manually after checking provider health | Bounded retry, jitter, circuit breaker, and dead-letter state |
| Maximum tool rounds | Trace ends with `max_rounds` | Review evidence and continue manually | Return a structured partial result with uncertainty |
| Prometheus query failure | Missing metric is silently skipped | Check observability health and use Kubernetes evidence | Attach per-source status and require minimum evidence |
| Loki failure | Logs are omitted | Use current/previous pod-log tools | Show explicit degraded banner |
| Tempo unavailable | No practical change because trace fetch is not implemented | Use metrics/logs and manual tracing | Implement and test trace adapter |
| Kubernetes tool timeout | Tool error is returned to model | Retry a specific read or stop analysis | Per-tool retry budget and dependency metrics |
| Namespace concurrency full | New analysis is rejected as rate limited | Retry when a slot frees | Durable priority queue with fairness |
| Persistent volume full/unwritable | Artifact writes fail; some state may remain only in memory/UI | Stop new analyses and recover capacity | Storage readiness gate and remote durable store |
| Slow SSE client | Events can be dropped | UI polling refreshes current state | Replayable event sequence or managed stream |
| Backend restart | In-memory streams and running analysis are lost | Re-open issue and rerun analysis | Durable job state and resumable workers |

## Back pressure, retry, and idempotency

Current controls include a maximum of 100 alerts per webhook payload, five
concurrent analyses per namespace by default, a configurable tool-round limit,
and HTTP timeouts. Issue upsert by dedupe key prevents duplicate rows for repeat
alerts, and report helpers can detect prior fingerprint output.

The platform does not have a durable analysis queue. A production design should
introduce:

1. an idempotency key based on fingerprint and analysis version;
2. priority-aware durable queueing;
3. bounded retries by dependency and error class;
4. a terminal dead-letter state visible to operators;
5. leasing so a restarted worker can resume safely; and
6. a replay-safe result write.

## Availability and persistence

The Helm chart deploys one backend replica with a `Recreate` strategy and
ReadWriteOnce-style persistent storage. This avoids concurrent file writers but
creates a clear availability boundary during upgrades and node disruption.

The next maturity step is to move issue, job, trace, and audit metadata to a
transactional durable store; retain object storage for larger immutable
artifacts; and run stateless workers behind a queue. That enables multiple API
replicas without shared-filesystem races.

## Model and cost governance

Model choice should be based on scenario quality, latency, regional support,
and cost—not capability rank alone.

Current cost bounds are:

- up to 10 RCA tool rounds by default;
- 4,096 maximum output tokens per model call;
- 30 minutes of metric context;
- up to 50 initial log lines;
- bounded tool response sizes; and
- operator-started analysis rather than model calls for every incoming alert.

Missing controls are input/output token telemetry, per-analysis estimated cost,
daily/team budgets, caching, and model fallback policy.

Recommended selection process:

1. define the minimum acceptable quality/safety score;
2. compare candidate models on the same frozen scenarios;
3. select the lowest-cost model that clears the threshold and latency SLO;
4. reserve a stronger model for escalation when evidence is complex; and
5. alert on cost per useful, accepted RCA rather than raw token count alone.

## Security operations

Before production:

- use workload identity instead of static AWS keys;
- restrict diagnostic reads to approved namespaces at both application and RBAC
  layers;
- add log/label redaction before model submission and persistence;
- encrypt persistent artifacts and define access/audit ownership;
- protect webhook secrets from query logging and rotate them;
- integrate enterprise SSO and role-based authorization;
- enforce NetworkPolicy egress destinations;
- keep demo and chaos controls disabled; and
- review every Helm permissions change as a security-sensitive diff.

If remediation is enabled, alert on every approval, rejected action, execution,
and RBAC denial. A kill switch must disable write capability without removing
read-only diagnosis.

## Incident response for the AIOps platform

When the platform itself is degraded:

1. keep Alertmanager's existing receivers active;
2. treat dashboards and conventional runbooks as the fallback path;
3. stop new model analyses if evidence sources are unreliable;
4. disable remediation if approval, identity, or namespace controls are in
   doubt;
5. preserve traces needed to understand incorrect conclusions; and
6. communicate that an assistant outage does not block normal incident
   handling.

## Adoption plan

### Phase 1: shadow mode

- Remediation off.
- Operators run analyses after completing their own initial diagnosis.
- Reviewers score evidence, correctness, and usefulness.
- No AIOps output is used as the sole basis for action.

### Phase 2: assisted triage

- Operators may use the RCA draft during active response.
- Tool-level namespace enforcement and redaction are mandatory.
- Regression and cost dashboards are active.
- Teams maintain service-specific runbooks and scenario fixtures.

### Phase 3: selected approvals

- Enable only low-risk actions for selected namespaces.
- Require authenticated approval and recorded rollback plan.
- Review every execution outcome and false-positive proposal.

### Phase 4: scaled service

- Introduce durable queueing and highly available persistence.
- Integrate SSO, service ownership, and tenant policies.
- Establish model release governance and an operator feedback loop.

## Onboarding checklist

- Identify service owners and incident-response approvers.
- Register source namespaces separately from remediation namespaces.
- Map alerts to deterministic triage cases and runbooks.
- Add at least one reproducible failure scenario per critical service.
- Define sensitive labels and log fields to redact.
- Verify Prometheus and Loki connectivity; treat Tempo as unavailable until its
  adapter is implemented.
- Run the complete evaluation suite with the chosen Bedrock model.
- Practice model failure and platform fallback.
- Train operators to inspect AI Trace before accepting a conclusion.
- Capture acceptance feedback and rejected diagnoses.

## Roadmap

Prioritized engineering work is:

1. enforce read-tool namespace authorization and reduce Secret RBAC;
2. implement redaction and explicit evidence-source health;
3. wire alert playbook handlers only after durable queueing is available;
4. add retries, idempotency, and resumable job state;
5. implement Tempo and selected service connectors;
6. collect token, cost, and claim-to-evidence metrics;
7. add SSO, roles, and tenant-aware retention;
8. move persistence to a highly available design; and
9. evaluate managed agent-runtime capabilities only where they simplify
   operations without weakening tool and approval controls.

Predictive remediation and broader write automation remain roadmap items. They
should be considered only after diagnosis quality, auditability, and operator
trust are demonstrated over a representative incident sample.
