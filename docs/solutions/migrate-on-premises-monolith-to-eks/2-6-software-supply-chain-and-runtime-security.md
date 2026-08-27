# Software Supply Chain and Runtime Security

> **Phase:** 2 Design · **Category:** Modernization
> **Priority topic:** 22
> **Evidence standard:** Build, dependency, image, and manifest scanning
> patterns are retained. A complete production signing, admission, and runtime-
> detection estate is a target architecture unless separately evidenced.

## Abstract

Security controls cover the path from source to a running process. Passing one
scanner is not proof that source, dependencies, build service, registry,
deployment identity, runtime configuration, and node are all trusted.

## Control chain

| Stage | Required controls | Release evidence |
|---|---|---|
| Source | Protected branches, review, secret detection, dependency policy | Approved commit and exception record |
| Build | Isolated runner, pinned tooling, minimal permissions, no reusable production credentials | Build identity and immutable output digest |
| Artifact | SBOM, vulnerability scan, signing or attestation, retention policy | Digest, SBOM, scan disposition, provenance |
| Registry | Encryption, lifecycle, replication where required, scoped push/pull | Registry policy and pull test from target account |
| Admission | Approved registries, signatures, Pod Security, policy as code | Allowed and denied admission tests |
| Runtime | Non-root, read-only filesystem where compatible, minimal capabilities, network policy, detection | Effective Pod spec and representative runtime alert |
| Response | Image quarantine, credential rotation, rollback and forensic retention | Exercised response record |

## Implementation sequence

1. Create a release threat model and define what blocks promotion.
2. Produce one immutable image digest and SBOM from a controlled build.
3. Scan source, dependencies, Dockerfile, filesystem, image, IaC, and rendered
   Kubernetes manifests; deduplicate findings by affected release.
4. Sign or attest the release and verify it before admission.
5. Apply Pod Security Standards and policy checks in audit mode before enforce.
6. Restrict egress, Linux capabilities, host access, privilege escalation, and
   writable paths according to workload requirements.
7. Connect cloud, Kubernetes audit, network, and runtime findings to a tested
   incident process.

## Validation and rollback

- a tampered or unsigned image is rejected while an approved digest deploys;
- an intentionally privileged or non-compliant Pod is rejected;
- blocking findings have an owner, deadline, compensating control, and approval;
- runtime detections identify the workload, node, image, and response owner;
- enforcement begins with a canary namespace and documented emergency bypass;
- rollback reverts the policy revision, not the evidence or audit trail.

Availability claims must account for admission-webhook failure modes. A
security control that can stop every deployment requires high availability,
bounded timeouts, explicit failure policy, monitoring, and a rehearsed recovery
path.

## References

- [Amazon EKS Pod Security best practices](https://docs.aws.amazon.com/eks/latest/best-practices/pod-security.html)
- [Kubernetes security practices](../../practices/k8s/security.md)
