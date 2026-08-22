# Amazon EKS Best Practices：Security

EKS security 是 shared responsibility。AWS 保護 managed control plane，platform team 則負責 cluster access、worker nodes、workloads、data、network paths，以及 detection and response。應採用 defense in depth，避免單一 misconfiguration 暴露 cluster。

> 最後檢視：2026-08-20

## Identity、workload 與 runtime protection

- Human access 使用 AWS IAM Identity Center 與 EKS Cluster Access Management (CAM)，透過 roles/groups 授權，移除永久 `cluster-admin`，並定期 audit access entries。
- 優先使用 private cluster endpoint；若必須 public access，限制 CIDRs 並保留 nodes/operators 的 private endpoint access。
- 每個 workload 使用 dedicated Kubernetes service account 與 AWS IAM role；新 workload 在支援的 node types 上優先使用 EKS Pod Identity，並將 associations 與 trust policies 限定至 cluster、namespace 與 service account。
  - 共用 identity 會讓任一 compromised pod 取得其他 workload 的 AWS permissions，也使 CloudTrail attribution 失去辨識度。
  - EKS Pod Identity 不需為每個 cluster 建立 OIDC provider，支援 session tags 且不消耗 account STS quota；AWS 建議在可行時優先使用。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html)
  - 已有成熟 OIDC/IRSA pattern、使用 Fargate 或 Windows nodes，或需要直接 `AssumeRoleWithWebIdentity` cross-account federation 時，可繼續使用 IRSA。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/multi-account-strategy.html)
- IAM 與 Kubernetes RBAC 均採 least privilege，避免 wildcard permissions、廣泛 `ClusterRoleBinding`、長期 service-account tokens 與不必要 anonymous API access。
  - IAM 控制 AWS API，RBAC 控制 Kubernetes API；只縮限其中一層，仍可能留下另一條 privilege-escalation path。
- 不需要呼叫 Kubernetes API 的 pods 停用 automatic service-account token mounting；限制 EC2 instance metadata，確需使用時採 IMDSv2。
- 以 Pod Security Standards/Pod Security Admission 或 policy-as-code engine enforce；containers 以 non-root 執行、移除不必要 Linux capabilities、使用 seccomp、避免 privileged/host namespace access，並在可行時使用 read-only root filesystem。
  - 這些 controls 用來限制 container escape 後的能力與 host attack surface；應由 admission policy 阻擋違規 deployment，而非只靠人工 review。
- 使用 approved minimal base images，scan images/dependencies、產生 SBOM、簽署 trusted artifacts，且只允許 approved registries 的 verified images。
- 保持 nodes 與 managed add-ons patched；優先 immutable replacement，降低 host access，必要時用 dedicated nodes 或 stronger sandboxing 隔離 sensitive workloads。
  - EKS 已於 2025-11-26 停止發布 EKS-optimized AL2 AMI，且 Amazon Linux 2 已於 2026-06-30 EOS；production nodes 應遷移至 AL2023 或 Bottlerocket，不應繼續把 AL2 視為可取得安全更新的 baseline。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/userguide/eks-ami-deprecation-faqs.html)

## Network、data、tenant isolation 與 response

- 以 default-deny ingress/egress network policies 開始，明確允許 DNS，再逐步加入必要 flows；在 enforcement 前後監控 policy decisions。
- 需要 AWS resource-level controls 時使用 security groups for pods，使用 Kubernetes network policies 做 pod/namespace segmentation；兩者是 complementary controls。
- 依 threat model 在 load balancers 與 services 間加密 traffic；透過 ACM、private CA 或 service mesh 管理 certificates。
- 使用 AWS KMS 加密 persistent data 與 Kubernetes secrets，rotate keys/secrets，優先 external secrets provider，能以 files mount 時不要放在 environment variables。
- Soft multi-tenancy 結合 namespaces、RBAC、quotas、network policies、admission policy 與 scheduling isolation；需要 hard boundary 時使用 separate clusters 或 AWS accounts。
  - Namespace 不是完整 security boundary；若 tenants 互不信任、法規要求隔離或 blast radius 必須獨立，應提升到 cluster/account boundary。
- 啟用適合 monitoring/retention policy 的 EKS API、audit、authenticator、controller-manager 與 scheduler logs，集中 control-plane、node、workload、network-policy 與 cloud audit signals。
- 啟用 Amazon GuardDuty Runtime Monitoring 或具等效能力的 runtime security solution，偵測 container process、file access、network connection、credential abuse 與 privilege escalation。
  - GuardDuty 可結合 Kubernetes audit logs、CloudTrail、VPC Flow Logs、DNS 與 runtime events 產生 findings；Runtime Monitoring 支援 EC2 與 Auto Mode，但不支援 Fargate 或 Hybrid Nodes。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/runtime-security.html) [AWS 官方文件](https://docs.aws.amazon.com/guardduty/latest/ug/how-runtime-monitoring-works-eks.html)
- Incident runbook 應能 identify pod/node、isolate traffic、revoke credentials、cordon/preserve node、capture volatile evidence 並 redeploy clean workloads；在終止 compromised infrastructure 前保留 evidence。
  - 立即刪除 node 可能同時清除 process、memory、network connection 與 local log evidence，讓 root-cause analysis 無法完成。

## Review checklist

- Human 與 workload identities 分離、短期有效且 least-privileged。
- Public API access、privileged pods、host access 與 wildcard permissions 有 documented justification。
- Default-deny network policy 與 encryption 覆蓋 sensitive workloads。
- Images、nodes、add-ons、secrets 與 KMS keys 都有 update 或 rotation owners。
- Security logs 集中保留，incident runbook 已測試。

---
# Amazon EKS Best Practices: Security

EKS security is a shared responsibility. AWS secures the managed control plane,
while the platform team remains responsible for cluster access, worker nodes,
workloads, data, network paths, and the detection and response process. Apply
defense in depth so that one misconfiguration does not expose the cluster.

## Identity and access

- Use AWS IAM Identity Center with EKS Cluster Access Management (CAM) for
  human access. Grant access through roles and groups, remove permanent
  `cluster-admin` access, and audit access entries regularly.
- Prefer a private cluster endpoint. If public access is required, restrict its
  CIDRs and retain private endpoint access for nodes and in-VPC operators.
- Give each workload a dedicated Kubernetes service account and AWS IAM role.
  Prefer EKS Pod Identity or IAM Roles for Service Accounts (IRSA) over node
  credentials, and scope trust policies to the intended cluster, namespace,
  and service account.
- Apply least privilege to both IAM and Kubernetes RBAC. Avoid wildcard
  permissions, broad `ClusterRoleBinding` objects, long-lived service-account
  tokens, and unnecessary anonymous API access.
- Disable automatic service-account token mounting when a pod does not call the
  Kubernetes API. Restrict access to EC2 instance metadata and use IMDSv2 when
  a workload genuinely needs it.

## Workload and runtime protection

- Enforce the Kubernetes Pod Security Standards with Pod Security Admission,
  or use a policy-as-code engine when rules need more context and customization.
- Run containers as non-root, drop unneeded Linux capabilities, use seccomp,
  avoid privileged and host namespace access, and make root filesystems
  read-only where the application permits it.
- Use approved minimal base images, scan images and dependencies, generate an
  SBOM, sign trusted artifacts, and admit only verified images from approved
  registries.
- Keep nodes and managed add-ons patched. Prefer immutable replacement over
  in-place host changes, minimize host access, and isolate sensitive workloads
  with dedicated nodes or stronger sandboxing where required.
- Use GuardDuty or an equivalent runtime control to detect suspicious API,
  network, credential, and container behavior.

## Network, data, and tenant isolation

- Begin with default-deny ingress and egress network policies, explicitly allow
  DNS, and then add only the flows each workload needs. Monitor policy decisions
  before and after enforcement.
- Use security groups for pods when AWS resource-level controls are needed, and
  Kubernetes network policies for pod and namespace traffic. They are
  complementary controls.
- Encrypt traffic at load balancers and between services when the threat model
  requires it. Manage certificates through ACM, a private CA, or a service mesh
  rather than embedding certificates in images.
- Encrypt persistent data and Kubernetes secrets with AWS KMS. Rotate keys and
  secrets, prefer an external secrets provider, and mount secrets as files
  instead of exposing them in environment variables when possible.
- For soft multi-tenancy, combine namespaces, RBAC, quotas, network policies,
  admission policy, and scheduling isolation. Use separate clusters or AWS
  accounts when tenants require a hard security boundary.

## Detection and response

- Enable EKS API, audit, authenticator, controller-manager, and scheduler logs
  according to the cluster's monitoring and retention policy.
- Centralize control-plane, node, workload, network-policy, and cloud audit
  signals. Alert on unexpected access, privilege changes, policy failures,
  suspicious runtime activity, and disabled controls.
- Prepare an incident runbook that can identify the affected pod and node,
  isolate pod traffic, revoke credentials, cordon and preserve the node, capture
  volatile evidence, and redeploy clean workloads.
- Practice game days and test the response workflow before an incident. Preserve
  evidence before terminating compromised infrastructure.

## Review checklist

- Human and workload identities are separate, short-lived, and least-privileged.
- Public API access, privileged pods, host access, and wildcard permissions have
  documented justification.
- Default-deny network policy and encryption controls cover sensitive workloads.
- Images, nodes, add-ons, secrets, and KMS keys have update or rotation owners.
- Security logs are retained centrally and the incident runbook is tested.
