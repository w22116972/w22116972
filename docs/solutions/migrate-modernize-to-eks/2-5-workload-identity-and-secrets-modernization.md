# Workload Identity and Secrets Modernization

## Role in the journey

**Category:** Modernization
**Priority topic:** 21
**Evidence status:** The retained solution supports per-workload identity and
external Secret references. EKS Pod Identity adoption remains a target decision
unless runtime evidence is retained for a specific workload.

The objective is to replace node-wide permissions and long-lived credentials
with short-lived, workload-scoped identity while keeping secret values outside
images, Git, Helm values, and pipeline output.

## Identity decision

| Option | Prefer when | Important constraint |
|---|---|---|
| EKS Pod Identity | Supported SDKs and node types are used; simpler per-cluster association management is valuable | Requires the Pod Identity Agent and direct role association in the cluster account; cross-account access uses role chaining |
| IAM Roles for Service Accounts | Existing OIDC trust is established or direct cross-account web identity is required | Every cluster needs an OIDC provider and exact trust-claim controls |
| Node IAM role | Only for node-level components that cannot use a workload identity | Permissions are visible to a broader workload boundary and must be minimized |
| External secret authority | Values need independent rotation, audit, and access control | The synchronization or retrieval component becomes a security-critical dependency |

## Target contract

```text
pod
  -> dedicated Kubernetes ServiceAccount
    -> Pod Identity association or IRSA trust
      -> least-privilege IAM role
        -> approved AWS API

application
  -> Secret reference
    -> external secret authority
      -> short-lived or rotated runtime value
```

## Migration sequence

1. Inventory every AWS API call, Kubernetes permission, secret, certificate, and
   node-role dependency for the workload.
2. Create a dedicated ServiceAccount and least-privilege IAM policy.
3. Choose Pod Identity or IRSA from account, SDK, node, and cross-account needs.
4. Add the association and secret reference without removing the old path.
5. Validate required calls, denied calls, credential expiry, rotation, and audit
   attribution in a lower environment.
6. Remove static credentials and excess node-role permissions only after the
   new identity is observed in the effective workload.
7. Scan images, rendered manifests, logs, and Git history for leaked values.

## Production gates

- the running Pod uses the intended ServiceAccount and role;
- required actions succeed and representative unauthorized actions fail;
- application behavior survives credential refresh and secret rotation;
- human, node, controller, and workload roles are distinguishable in audit logs;
- break-glass access is time-bound, reviewed, and not embedded in automation;
- rollback restores the previous authorization path without restoring a leaked
  or expired credential.

## References

- [Amazon EKS identity and access management best practices](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
