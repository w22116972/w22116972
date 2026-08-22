# Migration Implementation

## Delivery principle

Each phase produces a deployable, testable, and reversible increment. Platform
foundation, application extraction, data change, traffic movement, and legacy
retirement are separate approvals because their rollback mechanisms differ.

## Phased delivery plan

| Phase | Inputs | Change | Validation | Rollback | Accountable owner | Completion evidence |
|---|---|---|---|---|---|---|
| 1. Inventory and baseline | Workload, dependency, traffic, resource, release, and incident data | Classify each capability and freeze measurement definitions | Owners confirm map, unknowns, and baseline window | No runtime change | Migration lead | Signed scope, dependency map, evidence register |
| 2. Platform foundation | Approved target architecture and account boundaries | Provision VPC/EKS/IAM/capacity and required controllers through IaC | Plan review, cluster access, node/add-on health, denied/allowed identity tests | Revert bounded IaC change or restore prior foundation state | Platform lead | Applied revision, inventory, platform test report |
| 3. Containerization | First-wave source, runtime and configuration inventory | Produce minimal immutable image and externalize configuration | Unit tests, local container test, vulnerability gates, non-root/read-only checks where compatible | Redeploy legacy artifact; no traffic moved | Service owner | Image digest, build record, scan disposition |
| 4. Release contract | Image plus service requirements | Add reusable Helm chart, environment values, health, resources, policy, and rollback revision | Render, lint, schema/policy checks, lower-environment rollout | `helm rollback` or redeploy previous pinned revision | Service and platform owners | Chart version, rendered diff, ready rollout |
| 5. Service extraction | Dependency seam and compatibility plan | Route one capability to a new service; retain compatible old path | Contract, integration, data, load, and failure tests | Route back; keep schema backward compatible | Application lead | Parity report and approved wave record |
| 6. Traffic migration | Passed lower-environment and canary evidence | Shift a bounded route, tenant cohort, or traffic percentage | Real request-path probes plus metrics, logs, traces, and dependency health | Restore previous route/weight and image/chart revision | Release owner | Cutover record, observation report, decision |
| 7. Retirement and handoff | Stable observation window and no rollback dependency | Remove legacy route/component after archive and owner approval | No traffic, no consumers, recovery artifact, operator exercise | Reinstate retained artifact during agreed recovery period | Product owner and operations | Retirement approval and handoff sign-off |

## Foundation as code

Infrastructure code is organized by lifecycle and blast radius. The excerpt is a
sanitized pattern, not a copy of an internal root:

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name       = "cluster-a"
  kubernetes_version = var.kubernetes_version
  private_subnets    = module.network.private_subnet_ids

  managed_node_groups = {
    baseline = {
      min_size       = 3
      max_size       = 9
      instance_types = var.approved_instance_types
    }
  }
}
```

The version, sizes, and instance choices are environment inputs. Production
promotion requires a reviewed plan, explicit state ownership, upgrade
compatibility, and a recovery path. Terraform success is not sufficient: the
delivery also checks effective cluster, add-on, node, storage, identity, and
workload state.

## Containerization pattern

Retained repositories demonstrate multi-stage builds across several runtimes.
The public pattern keeps build tooling out of the runtime image and runs with a
non-root identity when the application supports it:

```dockerfile
FROM example-build-image:1 AS build
WORKDIR /src
COPY dependency-files ./
RUN ./dependency-tool restore
COPY . .
RUN ./build-and-test

FROM example-runtime-image:1 AS runtime
RUN addgroup --system app && adduser --system --ingroup app app
WORKDIR /app
COPY --from=build --chown=app:app /src/output ./
USER app
ENTRYPOINT ["./service-a"]
```

Important decisions are enforced outside the snippet:

- pin and regularly refresh approved base images;
- exclude credentials, local artifacts, and test fixtures with `.dockerignore`;
- produce one immutable version and record its digest;
- scan dependencies, Dockerfile, filesystem, and resulting image;
- keep configuration outside the image; and
- set a writable temporary path explicitly if the root filesystem is read-only.

## Helm release contract

Every service chart uses the same minimum contract while allowing workload
specifics:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  template:
    spec:
      serviceAccountName: service-a
      containers:
        - name: app
          image: registry.example.internal/service-a@sha256:example
          readinessProbe:
            httpGet:
              path: /ready
              port: http
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
```

The values are illustrative, not historical measurements. A production chart
also defines labels, ports, configuration and Secret references, security
context, graceful shutdown, availability controls, telemetry, and autoscaling
where evidence justifies them.

Health endpoints have different meanings:

- startup: initialization can finish;
- readiness: this replica can safely receive traffic; and
- liveness: the process cannot recover without restart.

A downstream outage should not automatically make every replica fail liveness
and amplify the incident.

## CI/CD quality gates

The retained source shows per-component pipelines that build images and deploy
Helm releases. The target flow tightens that mechanism into explicit promotion
and proof:

```yaml
stages: [test, build, scan, render, deploy, verify]

render:
  stage: render
  script:
    - helm lint ./helm_chart
    - helm template service-a ./helm_chart -f values/namespace-a.yaml > rendered.yaml
    - policy-check rendered.yaml

deploy:
  stage: deploy
  when: manual
  script:
    - helm upgrade --install service-a ./helm_chart --atomic --wait

verify:
  stage: verify
  script:
    - verify-rollout service-a
    - run-contract-tests service-a
    - run-synthetic-journeys service-a
```

Credentials are provided by protected identity or secret mechanisms, never
printed or copied into public artifacts. A production pipeline pins its tooling,
records image and chart identity, separates deploy authorization from build,
and retains enough metadata to identify the running source revision.

## Service extraction sequence

Extraction order is based on coupling and risk rather than visibility:

1. Start with a capability that has a clear API or asynchronous boundary, low
   data ownership risk, and observable business behavior.
2. Characterize the existing contract, including error and timeout behavior—not
   only successful responses.
3. Add a compatibility seam in the monolith so old and new implementations can
   coexist.
4. Build the service with an independent image, chart, owner, telemetry, and
   rollback revision.
5. Run the same contract and representative workload against both paths.
6. Move one route, tenant cohort, or bounded traffic slice.
7. Observe through a representative business cycle before expanding.
8. Remove the old implementation only after consumers, rollback, and data
   dependencies are cleared.

Later waves can use the now-proven platform pattern, but each service still owns
its specific data, latency, concurrency, and failure tests.

## Data and integration compatibility

Schema changes use expand-and-contract:

```text
add compatible schema
    -> deploy old and new readers
        -> backfill and reconcile
            -> switch bounded traffic
                -> observe
                    -> remove old field or path later
```

The rollback window must not depend on reversing an irreversible schema change.
For asynchronous flows, messages are versioned, consumers are idempotent, retry
limits are explicit, and poison messages have a recoverable destination. For
external integrations, timeouts, retry budgets, authentication, and rate limits
are included in contract tests.

## Configuration and secret migration

Configuration is classified before movement:

| Class | Example | Storage | Release behavior |
|---|---|---|---|
| Build-time invariant | Feature compiled into a specific artifact | Image metadata or source | New image required |
| Environment configuration | Endpoint, log level, feature flag | Versioned values or configuration object | Reviewed chart/config revision |
| Secret | Credential, key, token | External secret authority | Rotated independently; referenced by name |
| Dynamic operational setting | Rate or concurrency cap | Approved runtime configuration system | Audited change and rollback |

This prevents image rebuilding for environment changes and prevents secret
values from entering image layers, Helm values, command output, or Git history.

## Migration-wave evidence packet

Each wave retains:

- approved scope, owner, and dependency map;
- source, image digest, chart version, and rendered manifest;
- test and security-gate results with accepted exceptions;
- baseline and comparison query definitions;
- deployment and effective runtime state;
- cutover start/end, traffic scope, and observation window;
- threshold decision, incident notes, and rollback result; and
- handoff, runbook, dashboard, and follow-up owners.

Without this packet, a successful deployment is an event—not a defensible
migration outcome.
