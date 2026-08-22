# Amazon EKS 的 IAM Roles for Service Accounts（IRSA）

IAM Roles for Service Accounts（IRSA）可讓 EKS 工作負載透過 Kubernetes
ServiceAccount 取得暫時性 AWS 憑證。當工作負載身分機制可用時，不應將長期
AWS Access Key 放入應用程式碼、容器映像、Helm values、Kubernetes Secret
或 CI/CD 變數。

> **名詞說明：** 正確縮寫是 **OIDC**（OpenID Connect），不是 OICD。IAM
> OIDC provider 是 IRSA 的信任元件，不是與 IRSA 或 EKS Pod Identity 並列的
> 第三種工作負載身分方案。

## 如何選擇 IRSA 或 EKS Pod Identity

AWS 目前建議大多數新的 EKS 工作負載優先採用 **EKS Pod Identity**。EKS Pod
Identity 與 IRSA 都能提供暫時性憑證、工作負載層級的 IAM 權限、憑證隔離與
稽核能力；兩者都需要應用程式使用支援該功能的近期 AWS SDK，並採用預設憑證
提供者鏈。

### 功能與取捨比較

| 項目 | IRSA | IAM OIDC provider | EKS Pod Identity |
| --- | --- | --- | --- |
| 定位 | 將 Kubernetes ServiceAccount 對應至 IAM role 的工作負載身分方案 | IRSA 每個叢集所需的 IAM 信任錨點；本身不會授予工作負載 AWS 權限 | 由 EKS 管理 ServiceAccount 與 IAM role 關聯的工作負載身分方案 |
| 新工作負載建議 | 有特定需求時使用，仍受 AWS 完整支援 | 只有採用 IRSA 時才需要建立 | AWS 建議大多數新的 EKS 工作負載優先採用 |
| 叢集相依性 | 每個 role 的 trust policy 必須信任特定叢集的 OIDC issuer | 每個 EKS 叢集、每個使用 IRSA 的 AWS 帳戶都需要 provider | 不需要為每個叢集建立 IAM OIDC provider |
| IAM role 信任管理 | trust policy 必須包含 provider ARN、`aud` 與 `sub` | 必須管理 provider 生命週期及叢集替換 | role 信任 `pods.eks.amazonaws.com`，ServiceAccount 關聯由 EKS API 管理 |
| 跨帳戶存取 | 可直接以 `AssumeRoleWithWebIdentity` 信任另一帳戶中的 role | 目標帳戶也可能需要對應的 provider，視信任模式而定 | 只能直接關聯與叢集相同帳戶的 role；跨帳戶需使用 role chaining |
| 節點元件 | 不需要 EKS Pod Identity Agent | 不在叢集節點執行元件 | 需要在合格節點執行 EKS Pod Identity Agent add-on |
| STS 配額 | 使用該 AWS 帳戶的 STS 配額 | STS 會依 provider 與 token 驗證信任 | `AssumeRoleForPodIdentity` 不使用帳戶的標準 STS 配額 |
| ABAC | 不會自動提供 EKS Pod Identity 的 session tags | 不提供 workload session tags | 自動提供叢集、namespace、ServiceAccount 等 session tags，較適合大規模 ABAC |
| GitOps 可見性 | role annotation 可直接存在 ServiceAccount manifest | provider 通常由 IaC 管理 | 關聯是 EKS 資源，不只存在 Kubernetes manifest；GitOps 流程需同時管理雲端關聯 |
| 藍綠叢集替換 | 必須把新叢集 issuer 加入或更新每個 role trust policy | 新叢集需要新的 provider，完成遷移後移除舊 provider | role 信任通常可重用，但新叢集仍需建立 Pod Identity association 並部署 Agent |

### 選擇 IRSA 的情況

- 既有工作負載已穩定使用 IRSA，遷移效益不足以抵銷變更風險。
- EKS Pod Identity Agent 無法用於目標執行環境或組織不允許部署該元件。
- 需要直接使用 `AssumeRoleWithWebIdentity` 存取另一 AWS 帳戶的 IAM role。
- 使用的 EKS add-on、工具或既有 IaC 模組目前只支援 IRSA。

### 選擇 EKS Pod Identity 的情況

- 建立新的 EKS 工作負載，而且執行環境與 AWS SDK 都支援 Pod Identity。
- 希望避免在每個叢集建立及維護 IAM OIDC provider。
- 需要透過 session tags 實作 ABAC，或希望在大量叢集中重用較少的 IAM roles。
- 希望簡化藍綠叢集替換及集中管理 ServiceAccount 與 role 的關聯。

不要只因為 Pod Identity 較新就立即遷移所有穩定的 IRSA 工作負載。應先確認
Agent 支援範圍、SDK 版本、跨帳戶需求、IaC/GitOps 管理模式、回復方案及實際
權限測試。

## IRSA 的運作方式

1. EKS 叢集為 Pod 的 ServiceAccount 簽發短期、投射式的 OIDC token。
2. EKS admission webhook 將 `AWS_ROLE_ARN` 與
   `AWS_WEB_IDENTITY_TOKEN_FILE` 注入 Pod。
3. 支援 IRSA 的 AWS SDK 使用預設憑證提供者鏈，呼叫 AWS STS
   `AssumeRoleWithWebIdentity` 交換 token。
4. STS 回傳暫時性 role 憑證，AWS SDK 會在到期前自動更新。

IRSA 只處理 AWS IAM 授權。工作負載存取 Kubernetes API 的權限是另一個安全
邊界，必須透過 Kubernetes RBAC 控制。

## 前置需求

- 已建立 EKS 叢集，且操作者可存取其 Kubernetes API。
- 使用近期版本的 AWS CLI、`kubectl` 與 `eksctl`。
- 有權限建立 IAM OIDC provider、IAM policy 與 IAM role。
- 應用程式使用支援 Web Identity 憑證的 AWS SDK 版本。
- 應用程式採用 AWS SDK 預設憑證提供者鏈，不明確指定 node credentials。

以下範例使用這些佔位值：

```bash
export AWS_REGION="<region>"
export CLUSTER_NAME="<cluster-name>"
export NAMESPACE="<namespace>"
export SERVICE_ACCOUNT="<service-account-name>"
export ROLE_NAME="<iam-role-name>"
export POLICY_NAME="<iam-policy-name>"
```

## 1. 建立叢集的 IAM OIDC provider

EKS 叢集本身會提供 OIDC issuer URL，但 IRSA 還需要在 AWS 帳戶的 IAM 中建立
對應的 OIDC identity provider。兩者有關聯，但不是同一個資源。

取得 issuer ID，並確認 IAM provider 是否已存在：

```bash
OIDC_ID=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text | cut -d '/' -f 5)

aws iam list-open-id-connect-providers | grep "$OIDC_ID"
```

若沒有找到相符的 provider，建立一個：

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --approve
```

provider 的 audience 必須是 `sts.amazonaws.com`。若在私有 VPC 內無法解析
OIDC issuer hostname，可依 AWS 文件使用 EKS OIDC PrivateLink endpoint、從
AWS CloudShell 或可連線網際網路的環境執行，或設定 split-horizon DNS。

建立 provider 並不會自動授予任何 Pod AWS 權限；實際授權取決於 IAM role 的
trust policy 與 permissions policy。但 provider 是帳戶層級信任資源，仍應由
IaC 管理、加上可辨識叢集的 tags，並在叢集退役及所有 role trust 移除後刪除。

## 2. 建立最小權限 IAM permissions policy

只允許單一應用程式實際需要的 actions 與 resources。S3 bucket 層級與 object
層級 actions 必須使用不同的 resource ARN。以下範例只允許列出指定 prefix，
以及讀寫該 prefix 下的 objects：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListApplicationPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<bucket-name>",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["<application-prefix>", "<application-prefix>/*"]
        }
      }
    },
    {
      "Sid": "ReadWriteApplicationObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::<bucket-name>/<application-prefix>/*"
    }
  ]
}
```

將 policy 儲存為 `iam-policy.json`，然後建立 IAM policy：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-policy \
  --policy-name "$POLICY_NAME" \
  --policy-document file://iam-policy.json
```

除非應用程式確實需要，否則不要加入 `s3:DeleteObject`、萬用字元 actions 或
萬用字元 resources。視需求加入 KMS key 限制、來源帳戶、AWS Organizations
或其他 resource-policy 條件。

## 3. 建立具有嚴格 trust policy 的 IAM role

取得不含 `https://` 的 issuer hostname：

```bash
OIDC_PROVIDER=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text | sed 's#^https://##')
```

建立 `trust-relationship.json`，並將所有佔位值替換為實際文字：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<oidc-provider>:aud": "sts.amazonaws.com",
          "<oidc-provider>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

兩個條件都很重要：

- `aud` 將 token audience 限制為 AWS STS。
- `sub` 將 role assumption 限制為一個 namespace 中的一個具名
  ServiceAccount。

除非共用整個 namespace 是經過明確審查的設計決策，否則不要將 ServiceAccount
名稱替換成 `*`。預設應為每個應用程式建立專屬 IAM role 與 ServiceAccount。

建立 role 並附加 permissions policy：

```bash
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-relationship.json

aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn "arn:aws:iam::$ACCOUNT_ID:policy/$POLICY_NAME"
```

## 4. 建立並註記專屬 Kubernetes ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <service-account-name>
  namespace: <namespace>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<iam-role-name>
    eks.amazonaws.com/sts-regional-endpoints: "true"
automountServiceAccountToken: false
```

套用 manifest。AWS 建議使用 regional STS endpoint，以降低延遲、取得內建備援
並增加 session token 有效期。必須確認工作負載所在 Region 已啟用 STS。

當應用程式不需要呼叫 Kubernetes API 時，`automountServiceAccountToken: false`
可避免自動掛載 Kubernetes API token；IRSA 使用 webhook 注入的另一個 projected
web identity token。如果工作負載需要存取 Kubernetes API，才啟用 Kubernetes
token，並只透過 namespaced RBAC 授予必要權限；盡量避免萬用字元權限與
`ClusterRoleBinding`。

## 5. 將 ServiceAccount 指派給工作負載

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
  namespace: <namespace>
spec:
  template:
    spec:
      serviceAccountName: <service-account-name>
      containers:
        - name: application
          image: <immutable-image-reference>
```

新增或修改 ServiceAccount annotations 後必須建立新的 Pods，因為 admission
webhook 只會在建立 Pod 時注入 IRSA 設定。

## 6. 驗證設定

檢查 IAM provider、role trust、附加的 policies、ServiceAccount annotation 與
Pod 內注入的環境設定：

```bash
aws iam list-open-id-connect-providers
aws iam get-role --role-name "$ROLE_NAME" --query Role.AssumeRolePolicyDocument
aws iam list-attached-role-policies --role-name "$ROLE_NAME"

kubectl describe serviceaccount "$SERVICE_ACCOUNT" -n "$NAMESPACE"
kubectl exec -n "$NAMESPACE" <pod-name> -- env | grep '^AWS_'
kubectl exec -n "$NAMESPACE" <pod-name> -- aws sts get-caller-identity
```

caller identity 應為預期的 workload role，而不是 worker node role。若應用程式
容器未包含 AWS CLI，請透過應用程式的 AWS SDK，或使用相同 ServiceAccount 的
短期診斷 Pod 驗證。不要為了診斷而把 AWS CLI 永久加入正式映像。

## 安全與維運檢查表

- 新的相容工作負載優先評估 EKS Pod Identity；使用 IRSA 時記錄選擇原因。
- 使用近期、受支援的 AWS SDK 及預設憑證提供者鏈，並重用 SDK session，避免
  不必要的 STS 呼叫。
- 每個應用程式使用專屬 ServiceAccount 與 IAM role。
- 使用精確的 `aud` 與 `sub` 限制 trust policy；預設不要信任整個 namespace。
- IAM policy 與 Kubernetes RBAC 都必須遵守最小權限。
- 使用 regional STS endpoint，並確認該 Region 已啟用 STS。
- 防止 Pod 回退使用 node IAM role：限制 EC2 Instance Metadata Service（IMDS）
  存取、要求 IMDSv2，並將 node role 權限降至最低。變更 metadata hop limit 前
  先驗證 CNI、監控代理及其他 node workloads 的相容性。
- 不要建立靜態 ServiceAccount token Secrets。Kubernetes projected tokens 具有
  短期有效期，且會自動輪替。
- Kubernetes 同一個 Pod 內的 containers 通常共享該 Pod 的 AWS 身分邊界；不要
  把不受信任的 sidecar 與高權限應用程式放在同一個 Pod。
- 透過 AWS CloudTrail 監控 `AssumeRoleWithWebIdentity`，定期檢查 IAM Access
  Analyzer findings，移除未使用的 roles、policies 與 OIDC providers。
- 將藍綠叢集替換視為身分遷移：先加入新叢集 OIDC issuer、重新建立 Pods 並
  測試權限，確認流量切換後再移除舊 trust 與 provider。

## 參考資料

- [AWS：為叢集建立 IAM OIDC provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
- [AWS：將 IAM role 指派給 Kubernetes ServiceAccount](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)
- [AWS EKS 最佳實務：Identity and Access Management](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html)
- [AWS：讓 Kubernetes 工作負載存取 AWS](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html)
- [AWS：設定 ServiceAccount 使用的 STS endpoint](https://docs.aws.amazon.com/eks/latest/userguide/configure-sts-endpoint.html)
- [Kubernetes：Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes：RBAC good practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
