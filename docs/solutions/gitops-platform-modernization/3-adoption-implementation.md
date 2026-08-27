# Adoption Implementation

## Abstract

This phase covers adoption of existing Kubernetes objects, which Argo CD
performs by matching identity rather than through an import command. The first
objective is ownership transfer with no unintended runtime change, so the
evidence ladder, immutable-field behavior, and stateful examples are treated
as adoption risks rather than opportunities to upgrade or redesign.

## Adoption principle

Argo CD has no Terraform-style import command. It adopts an existing Kubernetes
object by matching its identity: API version, kind, namespace, and name. The
first objective is therefore ownership transfer with no unintended runtime
change—not an upgrade, cleanup, or redesign.

## Repeatable adoption workflow

1. **Inventory the live object.** Record owner references, managed fields,
   release metadata, image/chart versions, ServiceAccount, secrets mechanism,
   storage, generated children, external dependencies, and rollback owner.
2. **Locate every writer.** Identify Terraform, Helm, CI, bootstrap scripts,
   operators, and manual procedures that can mutate the same fields.
3. **Render the proposed desired state.** Use the existing release name and
   values. Pin versions and turn cluster-dependent Helm lookups into explicit,
   reviewed values where necessary.
4. **Compare desired with effective live state.** Normalize server defaults but
   do not dismiss material differences as formatting noise.
5. **Preserve identity and behavior.** Keep names, selectors, Service identity,
   workload identity, credentials references, PVCs, probes, and effective
   resources.
6. **Create the least-privileged AppProject and Application.** Start with manual
   sync, no self-heal, no prune, and no implicit namespace creation.
7. **Validate.** Run YAML/schema checks, Helm rendering, policy checks, and a
   server-side dry run against an appropriate live API server.
8. **Sync one adoption unit.** Observe Kubernetes rollout, UIDs, endpoints,
   controller behavior, storage, traffic, and external cloud resources.
9. **Transfer exclusive ownership.** Disable the prior writer only after the
   Argo-managed path is healthy and repeatable.
10. **Increase automation deliberately.** Enable self-heal after exclusive
    ownership; enable prune only after a dry-run prune review and an observation
    window. Record rollback and handoff evidence.

## Evidence ladder

| Stage | What it proves | What it does not prove |
|---|---|---|
| Source merged | Reviewed desired state exists | Pipeline execution, Argo observation, or deployment |
| CI passed | Selected render, schema, policy, and test checks passed | Live admission or runtime behavior |
| Helm revision rendered | Chart and values can produce manifests | Argo owns them or Kubernetes accepted them |
| Argo `Synced` | Live tracked fields match the desired revision | Healthy pods, traffic, data, or external side effects |
| Kubernetes rollout ready | Workload controller reached readiness | End-to-end service behavior or data correctness |
| Effective-state checks passed | Traffic, identity, storage, membership, and dependencies behave as intended | Future reliability without monitoring and ownership discipline |

## Application pattern

The initial adoption form is intentionally conservative:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-controller
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://example.invalid/platform.git
    targetRevision: main
    path: platform/controller
  destination:
    server: https://kubernetes.default.svc
    namespace: platform-system
  syncPolicy:
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=false
```

There is no `automated` block during the ownership transfer. Later automation
is a separate reviewed change, not a hidden side effect of registering the
Application.

## Stateless/controller adoption example

An existing load-balancer controller was installed by standalone Helm and
depended on Terraform-managed IAM and a pre-existing ServiceAccount. The
adoption kept the exact release name, chart/controller version, replica count,
cluster settings, and ServiceAccount reference. The chart was prevented from
creating a second identity.

Before sync, the team compared rendered CRDs, RBAC, Deployment, webhooks, and
TLS handling with live state. Generated certificate data and webhook CA bundles
were preserved through narrowly scoped diff rules; those rules did not ignore
the whole Secret or webhook specification. The first server-side sync was
manual and non-pruning.

Evidence after sync showed the controller replicas remained ready, admission
continued to work, and the existing load balancers and target groups were not
replaced. Only then was the Helm-writing bootstrap logic retired. A later,
separate change enabled self-heal and prune.

This example demonstrates why “the Deployment is healthy” is insufficient:
the effective checks included webhook admission, workload identity, generated
certificates, and absence of cloud-resource replacement.

## Stateful adoption example

A graph database consisted of four existing Helm-managed server StatefulSets
plus one separately managed shared Service. Adoption kept one manual
Application per server and a separate Application for the Service.

The chart used cluster lookups that Argo's renderer could not rely on. Without
explicit values, multiple Applications would render and claim the same shared
Service. The adopted design disabled that chart-generated Service, declared one
stable owner for it, referenced existing credential Secrets, and preserved
server names, selectors, storage classes, PVC claims, and advertised service
identity.

Each server was synced independently. Between syncs, operators checked pod
readiness, database membership, routing, and PVC binding. On 2026-08-19 all four
server Applications and the shared-Service Application were `Synced` and
`Healthy`; all four server pods were ready and their four existing PVCs remained
bound. These five Applications remain manual because a healthy previous sync
does not remove the need to sequence future membership-affecting changes.

Stateful adoption must stop when any of these are unknown:

- backup and restore behavior;
- immutable StatefulSet changes;
- PVC retention or reclaim policy;
- stable Service/selector identity;
- database quorum, membership, or topology;
- credential and license Secret ownership; or
- which member can be safely restarted first.

## Immutable fields and Helm lookup behavior

Adoption can reveal a mismatch that ordinary Helm upgrades hid. Service
ClusterIPs, selectors, volume claim templates, and parts of StatefulSet specs
cannot be changed in place safely. The response is not to force replacement.
Options include matching live state exactly, separating a shared object into its
own Application, retaining an externally owned object, or scheduling a distinct
migration with outage and recovery controls.

Charts that use `lookup` also need special attention. A direct Helm client can
read the cluster during render, while Argo's repository rendering is designed
to be deterministic. Values gated by lookup should be made explicit so that
the Git-rendered result is stable and reviewable.

## CI/CD controls

Pre-merge controls should include:

- YAML and repository convention checks;
- Helm lint and deterministic template rendering;
- Kubernetes schema and policy validation;
- tests for overlays, value layering, and ApplicationSet generation;
- image and chart version pinning;
- checks that Secrets contain references rather than values; and
- a rendered diff attached to review for high-risk Applications.

Server-side dry run belongs in an environment that has the relevant CRDs and
admission webhooks. Post-merge controls observe Argo revision and health, then
run workload-specific checks. The checked implementation retained a read-only
post-sync gateway verification job after retiring the CI jobs that directly
applied gateway manifests. This preserved operational evidence without leaving
a second writer.

## Promotion

Promotion advances an immutable chart, image, or Git revision through reviewed
environment configuration. It does not copy live objects from one cluster to
another. ApplicationSet keeps the Application shape consistent, while overlays
carry the deliberate environment differences.

Before promotion, record:

- tested revision and image digest;
- sync and rollout result in the source environment;
- material environment differences;
- database or cloud-side effects;
- rollback revision; and
- the owner observing the promotion window.

## Rollback mechanics

For an automated Application, the durable rollback is a Git revert or reviewed
fix-forward followed by reconciliation. For an immediate emergency change,
pause automation first so self-heal does not fight the operator.

For a manual stateful Application:

1. stop further syncs;
2. assess membership, storage, and data integrity;
3. restore the previous pinned desired state if the change is configuration
   only;
4. use the datastore recovery runbook if data was affected; and
5. verify membership, PVCs, readiness, and client behavior before proceeding to
   another member.

Deleting an Application is not a normal rollback. Finalizers and prune policy
can cascade into real workloads and cloud resources, so deletion requires the
same blast-radius review as deleting the underlying system.
