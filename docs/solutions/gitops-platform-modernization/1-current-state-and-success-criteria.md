# Current State and Success Criteria

## Scope and evidence standard

This case study covers the adoption of an existing Amazon EKS platform into an
Argo CD operating model. It is not an installation tutorial and it does not
claim that every application release moved to Argo CD.

The evidence model separates six states that are often incorrectly collapsed
into “deployed”:

1. desired configuration exists in source control;
2. validation or a pipeline job passed;
3. a Helm release or manifest revision was produced;
4. Argo CD reports the desired revision as `Synced`;
5. Kubernetes reports the workload as rolled out and ready; and
6. the effective service, storage, identity, and traffic behavior is correct.

The public facts below were rechecked on 2026-08-19 against the platform
repository and a read-only inventory of `cluster-a`. Names, locations, account
details, repository addresses, and workload identifiers are sanitized.

## Customer context

The platform hosted multiple environments, shared controllers, observability
components, gateways, and stateful data services. Delivery had accumulated
several legitimate but overlapping paths:

- Terraform for AWS infrastructure, Amazon EKS, IAM, and managed add-ons;
- bootstrap scripts and standalone Helm releases for cluster controllers;
- CI jobs that applied Kubernetes manifests directly;
- manual `kubectl` changes used during incidents; and
- application-team deployment pipelines outside the platform repository.

The modernization goal was not “put everything in Argo CD.” It was to give each
resource one lifecycle owner, adopt existing resources without recreating them,
and make the path from reviewed source to effective runtime state observable.

## Current-state risks

| Risk | Operational consequence |
|---|---|
| Two writers manage the same fields | CI, Helm, and Argo can repeatedly overwrite each other |
| A merged manifest is treated as deployment proof | Teams miss failed syncs, degraded rollouts, and ineffective configuration |
| Controller-generated objects are adopted directly | Argo competes with the controller or prunes required runtime resources |
| Automated prune is enabled during discovery | A wrong inventory can remove traffic-serving or stateful objects |
| Helm defaults replace effective live values | Adoption can change replicas, memory, webhooks, services, or storage |
| Stateful resources are batch-synced | Immutable fields, PVC identity, or database membership can be disrupted together |
| Secrets are copied into Git during adoption | Credential exposure becomes part of repository history |
| Emergency changes remain only in the cluster | Drift returns after reconciliation or is lost during recovery |

## Ownership inventory

This inventory records lifecycle ownership, not merely which tool can render an
object.

| Resource | Current owner before adoption | Desired owner | Stateful risk | Adoption method | Rollback | Evidence |
|---|---|---|---|---|---|---|
| VPC, EKS, node groups, IAM, OIDC | Terraform | Terraform | High | Import or align HCL, then require a reviewed plan | Restore prior state/configuration and apply reviewed plan | Terraform roots and plans |
| EKS managed add-ons | EKS through Terraform | EKS through Terraform | Medium | Keep outside Argo | Revert add-on version/configuration | EKS and Terraform inventory |
| Argo CD bootstrap | Bootstrap Helm/script | One-time bootstrap; live release managed by Argo | High | Install once, register root once, then use a manual self-management Application | Manual reviewed Helm/Argo rollback | Bootstrap script, root Application, live health |
| Platform controllers | Standalone Helm/script | Argo CD | High | Match existing release identity and values; manual first sync | Sync prior pinned chart or suspend Argo and restore old writer | Rendered diff, UID/readiness, controller behavior |
| Gateway and policy objects | CI `kubectl apply` | Argo CD | High | Diff and sync per environment, then retire the CI writer | Revert Git or disable automation before temporary repair | Argo status, Gateway conditions, route probes |
| Controller-generated data plane and cloud resources | Kubernetes/AWS controllers | Same controllers | High | Observe only; never adopt independently | Roll back the parent declaration | Owner references, cloud inventory, traffic |
| Observability services | Standalone Helm | Argo CD | Medium | Preserve release names, storage, identities, and values | Re-sync prior chart/value revision | Pod identity, readiness, PVC checks |
| Graph database servers and shared service | Manual Helm plus cluster fixes | Argo CD with manual sync | Critical | One server at a time; preserve names, selectors, credentials, PVCs, and membership | Stop sync, restore prior Git revision, recover from verified backup if data is affected | Argo health, membership, four bound PVCs |
| Workload credentials | Kubernetes or external secret workflow | External secret workflow | Critical | Reference existing Secret names; never import values into Git | Restore through credential owner | Secret references only, never Secret data |
| Product application releases | Application-team pipeline | Unchanged until separately approved | Varies | Explicitly excluded from platform adoption | Existing team rollback | Ownership register |

## Constraints

Adoption had to preserve:

- object identity: API group, kind, namespace, and name;
- Service names, selectors, ClusterIPs, and traffic behavior;
- ServiceAccounts, workload identity, and externally managed credentials;
- PVC names, storage classes, retention behavior, and database membership;
- controller webhooks, generated certificates, and controller-owned children;
- existing availability during the ownership transition; and
- a reversible path until the new reconciler had proved exclusive ownership.

Namespaces were intentionally pre-created rather than silently created by an
Application. This kept namespace lifecycle and tenant boundaries outside an
individual chart's blast radius.

## Success criteria

| Criterion | Acceptance test | Verified snapshot or target |
|---|---|---|
| Declared ownership | Every in-scope resource class names one reconciler and the previous writer is retired | Verified for adopted platform domains; product releases remain explicitly out of scope |
| Source convergence | Root and repository-backed Applications track the reviewed branch revision | Verified on 2026-08-19 |
| Argo convergence | Every Application is `Synced`; exceptions include an owner and next action | 31/31 `Synced` |
| Runtime health | `Healthy` plus workload-specific readiness and behavior checks | 30/31 `Healthy`; one documented image-pull exception |
| Safe automation | Self-heal and prune enabled only after zero/unambiguous diff and observation | 25/31 automated; six high-risk Applications deliberately manual |
| Stateful safety | PVC identity and database membership survive one-at-a-time adoption | Four database servers ready with four existing PVCs bound; shared service healthy |
| Secret boundary | Git stores references and policy, not secret values | Verified by source review; external secret implementation remains an owner responsibility |
| Environment consistency | New environment declarations are generated from a common pattern | One ApplicationSet generated five environment Applications in the checked cluster |
| Lead time | Measure merge-to-healthy duration using source, Argo, and rollout timestamps | Target; no defensible pre-adoption baseline exists |
| Rollback time | Revert a reviewed change and measure return to healthy effective state | Target; no comparable historical series exists |
| Recovery | Rebuild cluster state from Terraform, Git, bootstrap, and external secrets; restore data separately | Architecture defined; full disaster-recovery exercise remains a separate acceptance gate |
| Independent operation | Operators can onboard, diagnose drift, perform break glass, and reconcile back to Git without the delivery team | Handoff criterion, not claimed as measured completion |

## What is not claimed

The snapshot does not prove zero drift, shorter lead time, fewer incidents, or
full product-application adoption. `Synced` is not treated as `Healthy`, and
`Healthy` alone is not treated as evidence of correct traffic, identity, or data
behavior. Those limits are part of the operating model, not footnotes.
