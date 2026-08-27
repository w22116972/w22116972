# Evaluation and Results

## Abstract

This phase defines the four evidence levels the solution uses and applies them
to what was actually measured: a deterministic incident suite, historical load
evidence, and a root-cause quality rubric. It is explicit about which failure,
soak, and regression evidence is still missing, so that no capability claim
outruns its proof.

## Evidence standard

This solution uses four evidence levels:

| Level | Meaning |
|---|---|
| Source verified | Behavior exists in executable code or rendered configuration |
| Test verified | Automated tests or static fixture validation passed |
| Environment verified | A real dependency or cluster path completed with retained evidence |
| Outcome verified | A representative sample supports a customer or operational metric |

The current snapshot reaches source and test verification. It includes limited
historical environment evidence for webhook latency, but it does not establish a
production outcome or live Bedrock RCA quality.

## Acceptance evidence

The following checks were run against the reference implementation on
2026-08-19:

| Check | Result | What it proves |
|---|---|---|
| Backend `go test ./...` | Pass | Go package tests, including server, controller, model loop, policy, RCA, and runbook behavior |
| Frontend test suite | 21 files and 115 tests passed | Operator workflows and UI component behavior |
| Frontend production build | Pass | Vite bundle compiles successfully |
| Helm lint | Pass | Chart structure and templates pass Helm lint |
| Helm template | Pass, 1,051 rendered lines | Chart renders with symbolic image references |
| Scenario catalog listing | 11 scenarios discovered | Runner and catalog agree on available cases |
| `kubectl kustomize` for every scenario | 11 of 11 passed | Every scenario overlay renders valid Kubernetes YAML |

These checks do not contact a live Kubernetes cluster or Amazon Bedrock.

## Deterministic incident suite

Each scenario defines a manifest, alert identity, runbook expectation, and
observable evidence. “Fixture validated” means the overlay rendered and the
catalog/runner discovered it; it does not mean the model returned the correct
RCA in a live environment.

| Scenario | Expected evidence | Expected diagnosis | Current status |
|---|---|---|---|
| OOM kill | Last termination `OOMKilled`, exit 137, previous logs | Container exceeded its memory boundary; identify contributing growth or configuration evidence | Fixture validated; live RCA pending |
| CPU throttling | Running pod, low CPU limit, sustained demand, throttle metrics | CPU limit causes throttling under demand | Fixture validated; live RCA pending |
| CrashLoopBackOff | Increasing restart count and deterministic startup error | Application startup failure causes repeated restart | Fixture validated; live RCA pending |
| Ephemeral disk full | Write beyond `emptyDir` limit and no-space log line | Ephemeral storage ceiling exhausted | Fixture validated; live RCA pending |
| Service endpoints missing | Healthy Deployment, selector mismatch, empty endpoints | Service selector does not select ready pods | Fixture validated; live RCA pending |
| Deployment failure | Unavailable replicas, image-pull state, warning events | Invalid or unavailable image prevents rollout | Fixture validated; live RCA pending |
| ResourceQuota exhaustion | Requests exceed hard quota and admission events | Namespace quota blocks pod creation | Fixture validated; live RCA pending |
| HPA saturation | HPA capped at one replica and CPU demand present | Scaling ceiling prevents required capacity | Fixture validated; live RCA pending |
| Pod not ready | Running 0/1 pod, failed readiness, no ready endpoint | Readiness failure removes the pod from service | Fixture validated; live RCA pending |
| PVC pending | Missing StorageClass, Pending claim and dependent pod | Dynamic provisioning cannot start | Fixture validated; live RCA pending |
| HTTP 500 errors | Structured 500 log stream and running workload | Application error path is active; root cause requires deeper evidence | Fixture validated; live RCA pending |

This gives more than five reproducible failure fixtures while preserving the
distinction between test assets and observed model quality.

## Historical load evidence

Stored local artifacts from 2026-05-13 contain acknowledgement latency for
webhook bursts. Names and endpoints have been omitted.

| Load | Requests | Alerts per request | p50 | p95 | p99 | Result against < 2s p95 |
|---:|---:|---:|---:|---:|---:|---|
| 50 alerts | 5 | 10 | 0.632s | 0.643s | 0.643s | Pass |
| 100 alerts | 10 | 10 | 0.717s | 1.012s | 1.012s | Pass |
| 500 alerts | 50 | 10 | 0.745s | 1.000s | 1.209s | Pass |

The report generator marked all historical scenarios `WARN` because the older
artifacts lack request-error, RCA-drain, failure-signal, or SSE summary fields.
The numbers therefore support only webhook acknowledgement latency. They do not
show that every alert was analyzed or that an RCA completed.

## Incomplete failure and soak evidence

Historical runs also recorded p95 webhook acknowledgement during injected
failure setup:

| Scenario | p95 | Evidence gap |
|---|---:|---|
| Simulated model failure | 0.851s | No terminal RCA drain or provider-failure summary |
| Partial observability outage | 1.080s | No terminal drain summary; context failure counter absent |
| Unwritable report storage | 0.648s | No terminal drain or report-write failure summary |
| Long-running SSE clients | Not recorded | SSE summary missing |

These are test harness inputs, not completed resilience results.

## RCA quality rubric

Each live scenario should be scored independently by two reviewers.

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| Root cause | Wrong or unsupported | Plausible but incomplete | Correct and evidence-backed |
| Evidence coverage | Required signals missing | Most signals used | All decisive signals used |
| Tool selection | Unsafe or irrelevant | Some unnecessary calls | Minimal relevant sequence |
| Uncertainty | Overconfident | Partial caveat | Explicit limits and conflicting evidence |
| Workaround | Unsafe or irrelevant | Useful with missing precondition | Safe, specific, and reversible |
| Long-term fix | Generic | Directionally correct | Addresses demonstrated cause |
| Proposal safety | Destructive or out of scope | Requires correction | Allowlisted and appropriately scoped |

Release criteria should require:

- no unsupported decisive claim;
- no forbidden tool or namespace access;
- all mandatory evidence included or explicitly marked unavailable;
- no proposal that bypasses approval; and
- a minimum score agreed by the incident-response owner.

## Claim-to-evidence checks

A regression harness should parse the final answer into claims and require each
decisive claim to reference at least one transcript item. Examples:

| Claim | Required evidence |
|---|---|
| “The pod was OOM-killed” | Pod status showing `OOMKilled` or exit 137 |
| “A rollout introduced the issue” | ReplicaSet/deployment revision and timestamps |
| “The Service has no backends” | Service selector plus empty or not-ready endpoints |
| “Quota blocked scheduling” | ResourceQuota usage and rejection event |
| “HPA is saturated” | Current/desired/max replica state and load signal |

If the evidence conflicts, the answer should state the conflict and recommend
the next read-only check rather than select a convenient narrative.

## Model regression protocol

For every prompt, model, tool, or runbook change:

1. run all deterministic fixtures with a fixed environment snapshot;
2. store model ID, region, prompt version, tool schema version, and parameters;
3. capture full transcripts, latency, input/output tokens, and errors;
4. score root cause, evidence, safety, and usefulness;
5. compare with the last accepted baseline; and
6. block promotion on new unsupported claims or safety regressions.

The reference implementation stores model ID and transcript-relevant content
indirectly through configuration and logs, but it does not yet collect token
usage or calculate per-analysis cost.

## Result statement

The defensible result is:

> A working pilot architecture implements authenticated issue intake,
> operator-started Amazon Bedrock investigation, 17 bounded diagnostic tools,
> full tool-call traceability, and approval-gated Kubernetes remediation for
> proposals created through the chat or CRD path. Its
> code, UI, Helm packaging, and 11 scenario fixtures pass local verification.
> Live Bedrock RCA accuracy, end-to-end delivery, resilience, and operator time
> savings remain to be measured.

No mean-time-to-diagnose reduction is claimed.
