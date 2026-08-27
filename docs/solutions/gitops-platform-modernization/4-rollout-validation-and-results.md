# Rollout Validation and Results

## Abstract

This phase validates the modernization from source to effective state,
combining repository inspection with a read-only live check on a stated date.
It records the rollout sequence, the failures that changed the approach, the
measured outcomes, and the measurement contract behind them.

## Evidence standard

The modernization was validated from source to effective state. Repository
inspection established ownership declarations, Application configuration,
sync policy, CI behavior, and rollback intent. A read-only live check of
`cluster-a` on 2026-08-19 established the current Argo and Kubernetes snapshot.

Private infrastructure identifiers, URLs, revisions, logs, and credentials are
not reproduced here. Counts are aggregate evidence, and the snapshot is dated
because platform state can change after publication.

## Acceptance evidence

| Layer | Check | Result | Limitation |
|---|---|---|---|
| Source | Compare checked branch HEAD with repository-backed Argo revisions | Root and repository-backed Applications tracked the checked revision | Chart-sourced Applications report chart revisions instead |
| Argo definitions | Inventory Applications, AppProjects, ApplicationSets, and sync policies | 31 Applications, 12 AppProjects, one ApplicationSet | Counts describe one cluster snapshot |
| Argo sync | Inspect each Application's sync status | 31/31 `Synced` | Sync does not imply workload health |
| Argo health | Inspect health independently from sync | 30/31 `Healthy`; one `Progressing` | Health is controller-derived, not an end-to-end user test |
| Automation | Inspect automated, self-heal, and prune settings | 25/31 automated; six deliberately manual | One automated monitoring Application intentionally did not enable prune |
| Stateful database | Check Applications, StatefulSets, pods, PVCs, and shared Service | Four server Applications plus shared Service `Synced`/`Healthy`; four pods ready; four existing PVCs bound | This was a health check, not a restore drill |
| Stateful directory service | Check core StatefulSet, supporting Deployments, pods, and PVCs | Core StatefulSet 3/3 ready and three PVCs bound; one auxiliary Deployment unavailable | The upstream image-pull problem remained open |
| CI design | Inspect validation and mutation jobs | Server-side manifest validation remains; former direct gateway apply path is retired; post-sync verification is read only | A current remote pipeline run was not used as evidence |

## Rollout sequence

### Phase 1: establish boundaries

The team inventoried Terraform, bootstrap, Helm, CI, controller, and manual
writers. Cloud foundations, EKS managed add-ons, controller-generated children,
Secrets, and product release pipelines were explicitly excluded from Argo where
another owner remained correct.

### Phase 2: bootstrap a bounded control plane

Argo CD was installed through a one-time bootstrap. One root Application then
created domain AppProjects, child Applications, and an environment
ApplicationSet. The root automated only Argo custom resources; child workload
automation remained off during adoption.

### Phase 3: adopt low-risk declarations first

Gateway declarations and selected controllers were introduced as diff-only,
manual Applications. Rendered state was compared with live state, tracking-only
or explained differences were reviewed, and workload-specific checks ran after
sync.

### Phase 4: remove competing writers

After the Argo path proved healthy, bootstrap scripts became readiness checks or
IAM prerequisite installers, and CI stopped applying gateway resources. This
step created exclusive ownership; merely creating an Application would not have
done so.

### Phase 5: adopt stateful services conservatively

Stateful charts were matched to effective live configuration rather than stale
release defaults. Shared resources received a single owner. Database servers
were synchronized one at a time and remained manual after adoption.

### Phase 6: expand automation by blast radius

Self-heal was enabled after the observation window and prior-writer retirement.
Prune was enabled last, after dry-run review proved the Application would not
delete legitimate resources. Argo self-management and graph-database resources
remain manual by design.

## Failures and what they changed

### Client-side diff reported false drift

API-server defaulting made many gateway objects appear perpetually
`OutOfSync` under the original diff mode even though a server-side comparison
was empty. The platform enabled server-side diff consistently rather than
teaching operators to ignore noisy status. The lesson is to fix the comparison
model, not normalize alert fatigue.

### An AppProject destination blocked an otherwise merged Application

A backup Application was initially declared with a destination namespace that
its AppProject did not allow. Source existed, but Argo rejected the
specification before managing any child resource. Correcting the destination
restored sync. This was a useful failure: project boundaries caught a scope
error that a broad default project would have accepted.

### A successful sync did not prove a healthy deployment

The directory-service Application is currently `Synced` but `Progressing`.
Its three-server stateful core is ready and its three PVCs are bound, while one
auxiliary Deployment cannot pull its pinned upstream image. Git, rendering, and
Argo convergence cannot make an unavailable registry artifact runnable. Image
provenance and availability therefore belong in both pre-promotion validation
and runtime health checks.

### Stateful Helm lookup produced ambiguous ownership

A database chart used a live lookup before rendering a shared Service. Argo's
deterministic render could instead make every server Application claim that
Service. The solution was to disable the chart-generated copy and place the
Service under one explicit Application. Rendering the same chart was not enough;
ownership had to be redesigned.

## Outcomes

| Metric | Before | After / checked state | Evidence | Limitation |
|---|---|---|---|---|
| Platform ownership | Mixed bootstrap, Helm, CI, manual, and controller ownership | Explicit lifecycle matrix; one reconciler per adopted class | Ownership guide, source paths, retired writer jobs | Product deployments remain outside this migration |
| Argo inventory | Partial app-of-apps adoption | 31 Applications in 12 projects; one ApplicationSet generated five environment Applications | Live Argo inventory | One checked cluster only |
| Desired-state convergence | Not consistently visible | 31/31 Applications `Synced` | Live Argo status | Does not prove health |
| Runtime health | Required separate manual investigation | 30/31 `Healthy`; one named exception with effective workload evidence | Argo plus Kubernetes inventory | Not an application SLO measurement |
| Automated reconciliation | Disabled during adoption | 25/31 automated; six high-risk Applications manual | Live sync policies | Automation percentage is not a quality metric by itself |
| Gateway writer count | CI and emerging Argo path overlapped during transition | Argo is the sole gateway-manifest writer; CI retains read-only verification | Pipeline source and ownership docs | Does not cover product application writers |
| Stateful adoption | Manual Helm and cluster-local fixes | Four database servers and their shared Service tracked, healthy, and deliberately manual | Argo, pod, Service, and PVC checks | No claim of automated membership-safe upgrades |
| Environment declaration effort | Separate Application definitions | One overlay can be discovered by the ApplicationSet pattern | Repository structure and live generated Applications | Overlay content still needs review and testing |
| Deployment lead time | No comparable baseline | Measurement contract defined | Required timestamps documented | No reduction claimed |
| Rollback time | Inconsistently recorded | Git-revert and manual stateful paths documented | Runbook design | No measured reduction claimed |

## Measurement contract

Future claims use consistent timestamps:

- **lead time:** approved merge to Argo `Healthy` at the intended revision;
- **rollback time:** incident decision to restored effective behavior, not
  merely completion of a Git revert;
- **deployment failure:** a promoted change that fails sync, rollout, or its
  effective-state checks and requires intervention; and
- **operator effort:** active human minutes spent from alert to verified
  convergence.

Each measurement records environment, revision, Application, change class,
start/stop events, result, and excluded wait time. Without this contract,
percent-improvement claims would not be defensible.

## Result statement

The platform moved from mixed, overlapping Kubernetes delivery paths to a
bounded GitOps operating model for its adopted platform domains. The result is
reviewable ownership, continuous drift visibility, repeatable environment
declarations, and a staged path from manual adoption to automated
reconciliation. The strongest evidence is not the number of Applications; it
is the retirement of competing writers and the explicit refusal to automate
stateful and self-management paths beyond their verified safety.
