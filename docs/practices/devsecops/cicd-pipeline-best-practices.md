# CI/CD Pipeline Best Practices

## Purpose

Build one traceable path from a reviewed source change to verified runtime
behavior. A successful build or merge is not proof that the intended version is
running correctly.

## Practices

- Keep changes small, reviewable, and attributable.
  - Protect main branches, require peer review for material changes, and retain
    the source revision throughout build and deployment evidence.
- Build immutable artifacts once.
  - Promote the same signed image or package across environments instead of
    rebuilding it with environment-specific differences.
  - Generate a software bill of materials and attach provenance where the risk
    profile requires it.
- Apply layered gates according to risk.
  - Run formatting, unit tests, static analysis, secret scanning, dependency and
    image scanning, IaC validation, manifest rendering, and policy checks early.
  - Add integration, migration, performance, resilience, and security tests only
    where they exercise meaningful failure modes.
- Separate deployment from release.
  - Use rolling, canary, blue/green, or feature-control strategies according to
    rollback needs and state compatibility.
  - Require approval for destructive infrastructure, database, identity, or
    production changes.
- Verify effective state after deployment.
  - Confirm the deployed revision, rollout health, traffic, identity, storage,
    migrations, service objectives, and external side effects.
  - Automatically stop or roll back only when the rollback path is known to be
    safe.
- Preserve evidence and recovery paths.
  - Retain test results, plans, approvals, deployment events, and verification
    output with the change record.
  - Regularly exercise artifact rollback, configuration revert, database
    recovery, and break-glass access.

## Delivery flow

Use a short, repeatable path for every change. The exact branch names may vary,
but the purpose of each environment and gate should remain clear.

| Change | Required gates | Result |
| --- | --- | --- |
| Push to a feature branch | Build, unit tests, linting, static analysis, dependency vulnerability scanning | Fast feedback on a buildable, testable change |
| Pull request to `develop` | All feature-branch gates plus integration tests | Evidence that services and modules work together |
| Merge to `develop` | Build one immutable artifact, deploy it to staging, run end-to-end tests | A release candidate verified in a production-like environment |
| Pull request to `main` | All CI and staging gates; performance tests when they cover a meaningful regression risk | Approval-ready production candidate |
| Merge to `main` | Promote the verified staging artifact, deploy to production, monitor, and retain rollback evidence | Controlled production release |

Do not rebuild an artifact for a later environment. Promote the same immutable
artifact that passed the earlier gates, using environment-specific runtime
configuration only. Keep feedback fast by running the lowest-cost, highest-value
checks first, and add resilience, migration, or performance tests where the
change creates that specific risk.

## Application and infrastructure delivery

Keep application code and infrastructure configuration in separate repositories
when their ownership, permissions, or release cadence differ.

| Repository | Typical contents | Delivery responsibility |
| --- | --- | --- |
| Application code | Source code, Dockerfile, unit tests | Build, test, scan, and publish the immutable application artifact |
| Infrastructure configuration | Kubernetes manifests, Helm charts, and values files | Validate, approve, deploy, and verify the desired environment configuration |

The deployment record must link the source revision, artifact digest, and
infrastructure configuration revision. This makes it possible to prove what ran
in an environment and to roll back the relevant application or configuration
change safely.

## Shared pipeline templates and roles

Use a centrally maintained `cicd-templates` repository for standard pipeline
building blocks such as `build-docker.yml`, `deploy-k8s.yml`, and
`deploy-vm.yml`. Each service repository should use GitLab CI/CD `include` to
reference the appropriate templates, while keeping service-specific variables,
rules, and test commands close to the service.

The infrastructure team owns and maintains approved base images for application
teams. Base-image updates should be versioned, scanned, communicated, and
adopted through the same delivery gates as application changes.

## Evidence chain

```text
commit -> tests -> immutable artifact -> approved deployment
       -> runtime revision -> behavior and SLO checks -> release evidence
```

## References

- AWS Prescriptive Guidance, [CI/CD best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-cicd-litmus/cicd-best-practices.html)
- AWS Cloud Adoption Framework, [Continuous integration and continuous delivery](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-caf-platform-perspective/ci-cd.html)
