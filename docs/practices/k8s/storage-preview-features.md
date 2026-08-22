# Kubernetes Storage Preview Features 延伸指南

本文件隔離 Kubernetes v1.36 中仍為 Alpha 或 Beta 的 storage capabilities；production baseline 見 [Kubernetes Storage](storage.md)。Preview feature 的 API、feature gate、預設狀態與 provider 支援可能改變，不應只因官方文件已有範例就直接啟用。

## Projected volume preview sources

- `clusterTrustBundle` projection 在 v1.33 為 Beta，預設關閉。它把 signer 對應的 X.509 trust bundle 投射到 Pod；必須同時確認 API runtime configuration、feature gates、RBAC 與 bundle rotation。
- `podCertificate` projection 在 v1.35 為 Beta，預設關閉。Kubelet 透過 `PodCertificateRequest` 取得並輪替 private key 與 certificate；應驗證 signer policy、key storage、rotation failure、Pod restart 與 expiry monitoring。

這些功能處理 trust distribution 與 workload certificate lifecycle，不等同 Secret，也不應繞過外部 PKI 的 issuance、revocation 與 audit controls。

## Cross-namespace data sources

Custom volume populators 已於 v1.33 GA，保留在主指南。Cross-namespace volume data source 在 v1.26 為 Alpha，還需 `ReferenceGrant` 授權、相容 populator/controller 與對應 feature gates。採用前應驗證 cross-namespace ownership、grant revocation、confused-deputy risk、source deletion 與 upgrade compatibility。

## 採用 gate

1. 實際 minor version、API resources、feature gates、CSI driver 與 managed-provider support 全部已確認。
2. Preview API 只在 non-production 使用，並有 disable、rollback 與資料復原路徑。
3. 已測試 certificate rotation、controller restart、partial population、node drain 與 control-plane upgrade。
4. RBAC、admission、audit 與 metrics 能識別誰建立 source、誰讀取內容，以及失敗停在哪一層。

## 參考資料

- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Volume Populators and Data Sources](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Kubernetes v1.33: Volume Populators Graduate to GA](https://kubernetes.io/blog/2025/05/08/kubernetes-v1-33-volume-populators-ga/)

---

# Kubernetes Storage Preview Features Supplement

This guide isolates storage capabilities that remain Alpha or Beta in Kubernetes v1.36. See [Kubernetes Storage](storage.md) for the production baseline. Preview APIs, feature gates, defaults, and provider support can change; an example in the official documentation is not sufficient justification for production enablement.

## Projected volume preview sources

- The `clusterTrustBundle` projection is Beta in v1.33 and disabled by default. It projects X.509 trust bundles associated with a signer into a Pod; verify API runtime configuration, feature gates, RBAC, and bundle rotation together.
- The `podCertificate` projection is Beta in v1.35 and disabled by default. Kubelet obtains and rotates a private key and certificate through `PodCertificateRequest`; test signer policy, key storage, rotation failure, Pod restart, and expiry monitoring.

These features address trust distribution and workload-certificate lifecycle. They are not equivalent to Secrets and must not bypass external PKI issuance, revocation, and audit controls.

## Cross-namespace data sources

Custom volume populators reached GA in v1.33 and remain in the main guide. Cross-namespace volume data sources are Alpha in v1.26 and additionally require `ReferenceGrant` authorization, a compatible populator or controller, and the corresponding feature gates. Validate cross-namespace ownership, grant revocation, confused-deputy risk, source deletion, and upgrade compatibility.

## Adoption gate

1. Confirm the actual minor version, API resources, feature gates, CSI driver, and managed-provider support.
2. Use preview APIs only in non-production with disable, rollback, and data-recovery paths.
3. Test certificate rotation, controller restart, partial population, node drain, and control-plane upgrade.
4. Ensure RBAC, admission, audit, and metrics identify who creates a source, who reads its contents, and where failures stop.

## References

- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Volume Populators and Data Sources](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Kubernetes v1.33: Volume Populators Graduate to GA](https://kubernetes.io/blog/2025/05/08/kubernetes-v1-33-volume-populators-ga/)
