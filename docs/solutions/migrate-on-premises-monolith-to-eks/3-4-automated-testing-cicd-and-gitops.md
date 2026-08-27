# Automated Testing, CI/CD, and GitOps Modernization

> **Phase:** 3 Execute · **Category:** Modernization
> **Priority topic:** 15
> **Evidence standard:** CI-built images, Helm releases, deployment gates, and
> Argo CD patterns are retained. Adoption scope must be verified per workload; a
> Git commit alone is not runtime evidence.

## Abstract

CI proves the artifact. GitOps reconciles approved desired state. They are
separate control loops and should not both write the same Kubernetes objects.

## Target delivery model

```text
application pull request
  -> unit, contract, integration, and security tests
    -> immutable image and SBOM
      -> signed digest in registry
        -> environment configuration pull request
          -> review and policy checks
            -> Argo CD reconciliation
              -> rollout, dependency, traffic, and data verification
```

## Ownership boundary

| Layer | Writer | Required proof |
|---|---|---|
| Source and tests | Application team | Reviewed commit and test results |
| Image and SBOM | CI identity | Digest, provenance, and scan disposition |
| Environment desired state | Approved configuration pull request | Rendered diff and policy decision |
| Kubernetes application objects | Argo CD after adoption | Application health, sync revision, and effective object state |
| Platform foundation | Terraform or platform GitOps owner | Applied state plus AWS and cluster readback |
| Traffic approval | Release owner | Real request-path and service-objective evidence |

## Implementation sequence

1. Inventory the current pipeline, Helm identity, values, live objects, secrets,
   hooks, PVCs, and every active writer.
2. Select self-managed Argo CD or an available managed EKS capability from
   operational, regional, identity, feature, and support requirements.
3. Import one low-risk application with manual sync, no prune, and no self-heal.
4. Compare the full desired/live diff and preserve release names, immutable
   fields, ownership metadata, storage, and workload identities.
5. Sync, then validate workload, dependency, traffic, data, and rollback paths.
6. Retire the old deployment writer before enabling automated reconciliation.
7. Add prune and self-heal only after deletion scope, drift ownership, and
   recovery are tested.

## Validation and rollback

- a source revision maps to image digest, desired-state revision, Argo sync, and
  running digest;
- unauthorized direct mutation is detected without silently deleting valid
  emergency work;
- secrets are referenced, not stored as plaintext in Git;
- a Git revert restores safe desired state and is confirmed through live
  traffic;
- Argo controller failure does not stop already running applications;
- controller and repository recovery procedures are tested independently.

## References

- [Continuous deployment with Argo CD on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/argocd.html)
- [GitOps Platform Modernization](../gitops-platform-modernization/README.md)
- [CI/CD Pipeline Best Practices](../../practices/devsecops/cicd-pipeline-best-practices.md)
