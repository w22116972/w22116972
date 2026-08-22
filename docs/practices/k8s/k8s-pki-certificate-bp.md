# Kubernetes PKI 與憑證最佳實務

本文件適用於 kubeadm 或自行管理 control plane PKI 的 Kubernetes 叢集。Amazon EKS 等受管理服務通常由供應商管理 control-plane CA 與元件憑證；此時應遵循供應商的憑證與 endpoint 文件，不要嘗試修改受管理的 PKI。

## 建立信任邊界

- Kubernetes control plane 使用不同用途的 server、client 與 signing keys；不要把 TLS 憑證與 ServiceAccount signing key 視為同一類資產。
- 分離 Kubernetes general CA、etcd CA 與 front-proxy CA。只有使用 API aggregation layer 時才需要 front-proxy 憑證。
- CA private keys 優先保留在離線或專用簽署環境；若必須放在 control-plane node，應限制權限、加密備份並監控存取。
- Server certificate 必須包含實際使用的 DNS names 與 IP addresses 作為 SAN；不要依賴 Common Name 驗證主機名稱。
- etcd client、server 與 peer traffic 使用 mutual TLS，並限制網路存取。

## 使用最小權限身分

- 不要讓一般管理者或自動化使用 `system:masters`。該群組可繞過 RBAC，僅適合作為嚴格保護的 break-glass 身分。
- 日常管理使用受 RBAC 約束的群組，例如 kubeadm 的 `kubeadm:cluster-admins`，並為 controllers、nodes 與 users 使用不同憑證。
- Kubelet client certificate 的 CN 必須是 `system:node:<nodeName>`，Organization 必須是 `system:nodes`，且 node name 必須與註冊名稱完全一致。
- 避免共用 user certificate；每個人或 automation identity 應可獨立撤銷與稽核。

## 管理生命週期

- 建立 CA、component certificate、kubeconfig 與 ServiceAccount key 的 inventory，包括 issuer、SAN、用途、owner、到期日及儲存位置。
- 使用 kubeadm certificate management 或受管理的自動化進行 renewal；不要以未審核 script 手動覆寫 `/etc/kubernetes/pki`。
- 對短效 certificate 設定提前到期告警，並在 staging 練習 renewal、component restart 與 rollback。
- CA rotation 是多階段信任遷移：先發布新舊 CA bundle、重新簽發所有 dependents、驗證，再移除舊 CA。不可只替換單一檔案。
- 保護 `/etc/kubernetes/pki`、kubeconfig、etcd snapshot 與備份。確認 restore 後仍可取得需要的 keys、certificates 與 DNS/SAN。
- 不要將 private keys、含 embedded credentials 的 kubeconfig 或憑證備份提交到 Git、artifact store 或一般 log。

## 驗證

```console
kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -issuer -dates -ext subjectAltName
```

- 驗證 API server、etcd、kubelet 與 aggregated API 的完整 TLS chain、SAN、Extended Key Usage 及 clock synchronization。
- 憑證更新後確認所有 control-plane components、nodes 與管理 client 可重新連線。
- 定期測試撤銷或替換 compromised user、component 與 CA credentials 的程序。

## 參考資料

- [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Manual Rotation of CA Certificates](https://kubernetes.io/docs/tasks/tls/manual-rotation-of-ca-certificates/)

---

# Kubernetes PKI and Certificate Best Practices

This guide applies to kubeadm and other clusters where you manage control-plane PKI. Managed services such as Amazon EKS normally manage the control-plane CA and component certificates; follow the provider's certificate and endpoint guidance instead of modifying managed PKI.

## Establish trust boundaries

- Kubernetes uses server, client, and signing keys for different purposes. Do not treat TLS certificates and ServiceAccount signing keys as interchangeable assets.
- Separate the general Kubernetes CA, etcd CA, and front-proxy CA. Front-proxy certificates are needed only for the API aggregation layer.
- Prefer keeping CA private keys offline or in a dedicated signing environment. If a key must remain on a control-plane node, restrict access, encrypt backups, and audit use.
- Include every real DNS name and IP address in server-certificate SANs; do not rely on Common Name for hostname verification.
- Protect etcd client, server, and peer traffic with mutual TLS and network restrictions.

## Use least-privileged identities

- Do not use `system:masters` for routine administration or automation. It bypasses RBAC and should be a protected break-glass identity only.
- Use an RBAC-bound group such as kubeadm's `kubeadm:cluster-admins` for normal administration, with separate credentials for controllers, nodes, and users.
- A kubelet client certificate must use CN `system:node:<nodeName>`, Organization `system:nodes`, and the exact registered node name.
- Avoid shared user certificates. Each person and automation identity should be independently auditable and revocable.

## Manage the lifecycle

- Inventory every CA, component certificate, kubeconfig, and ServiceAccount key with its issuer, SANs, usage, owner, expiry, and storage location.
- Use kubeadm certificate management or reviewed automation for renewal; do not overwrite `/etc/kubernetes/pki` with ad hoc scripts.
- Alert before expiry and rehearse renewal, component restart, validation, and rollback in staging.
- Treat CA rotation as a staged trust migration: distribute a combined trust bundle, reissue dependents, validate, and only then remove the old CA.
- Protect `/etc/kubernetes/pki`, kubeconfigs, etcd snapshots, and backups. Confirm recovery includes the required keys, certificates, and DNS/SAN configuration.
- Never commit private keys, credential-bearing kubeconfigs, or certificate backups to Git, ordinary artifact storage, or logs.

## Validate

Use the commands above for kubeadm-managed clusters, then verify the complete TLS chain, SANs, Extended Key Usage, and clock synchronization for the API server, etcd, kubelets, and aggregated APIs. After renewal, confirm all components, nodes, and administrative clients reconnect. Regularly test replacement of compromised user, component, and CA credentials.

## References

- [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Manual Rotation of CA Certificates](https://kubernetes.io/docs/tasks/tls/manual-rotation-of-ca-certificates/)
