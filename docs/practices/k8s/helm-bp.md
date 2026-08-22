# Helm Chart 最佳實務

本指南協助建立容易設定、操作與安全升級的 charts。Chart templates、default values、dependency metadata 與 rendered manifests 都應視為 production code。

## 遵循 chart conventions

- Chart 名稱使用 DNS-1123 label：小寫字母、數字與 dash，首尾為字母或數字，最多 63 字元。
- Chart version 使用 Semantic Versioning；放入 Kubernetes label 時把 `+` 換成 `_`。
- YAML 使用兩個 spaces 縮排，不使用 tabs。
- 專案稱 `Helm`、command 稱 `helm`、一般名詞用 `chart`，檔名則是大小寫敏感的 `Chart.yaml`。
- Namespaced templates 不要 hard-code `metadata.namespace`；由 `--namespace` 或 GitOps configuration 決定部署 namespace。

## 為使用者設計 values

- 自訂 value 名稱以小寫字母開頭並使用 camelCase，例如 `servicePort`。
- 能簡化 template 時優先使用 flat values；多個相關設定形成清楚群組時才使用 nested maps。
- 需要以 `--set` 覆寫個別項目時優先使用 maps，不使用位置敏感的 lists。
- Strings 要 quote；必須原樣保留的大型 numeric identifier 應存為 string，再在 template 明確轉型。
- `values.yaml` 每個 property 都要文件化，comment 以完整 property name 開頭。
- Secrets 不得進入 committed values、`--set`、release metadata、CI logs 或 rendered manifests；改為引用外部管理的 Secret。
- 可行時使用 `values.schema.json`，在 installation 前拒絕錯誤 type、range 與 required value omission。

```yaml
# replicaCount is the desired number of application replicas.
replicaCount: 2

# image configures the application container image.
image:
  repository: "registry.example.com/team/service"
  tag: "1.4.2"
  pullPolicy: IfNotPresent

# service configures the Kubernetes Service.
service:
  port: 8080
```

## 維持 templates 可維護性

- Render YAML 的 template 使用 `.yaml`，只含 helpers 的檔案使用 `.tpl`。
- 檔名使用 dash 並包含 resource kind，例如 `api-deployment.yaml`；每個 resource definition 各自一檔。
- Defined template 全部 namespace，因為 chart 與 subcharts 的 helpers 是 global，例如 `{{ define "mychart.fullname" }}`。
- 使用兩個 spaces 縮排，template delimiters 內留空格；whitespace chomp 要有意識，並檢查 rendered output。
- 不應出現在 output 的實作註解用 Helm comments（`{{/* ... */}}`）；operators 應看到的才用 YAML comments。
- JSON syntax 只用於確實更易讀的小型 construct，複雜 structure 保持 YAML。
- Names、selectors 與 standard labels 集中在 `_helpers.tpl`，避免 resources 間 drift。
- Required input 以 schema validation 或 `required` 清楚失敗；有意識地使用 `quote`、`toYaml`、`nindent` 與 type conversion。

```yaml
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## 明確管理 dependencies

- 在 `Chart.yaml` 宣告 dependencies 並 commit 產生的 `Chart.lock`，讓 CI 與 developers resolve 相同版本。
- 使用合適的 Semantic Versioning constraint。Helm 指南建議 `~1.2.3` 這類 patch range；若 release policy 要求完全控制則使用 exact version。
- 優先使用 HTTPS repositories。`file://` 只適合固定 pipeline 組裝的 charts；downloader-plugin URL 要求所有使用者安裝相同 plugin。
- 每個 optional dependency 都提供如 `database.enabled` 的 condition；多個 subcharts 共同形成一個 optional feature 時可使用 shared tags。
- 修改 constraints 時有意識地執行 `helm dependency update`，並 review `Chart.lock` 與下載版本。

引用的 Helm dependency 頁面註明尚未針對 Helm 4 更新；version range 與 dependency behavior 應再依 delivery pipeline 的 Helm 版本確認。

## 使用一致的 metadata

Kubernetes 或 operators 需要查詢的 metadata 使用 labels；非識別 metadata 使用 annotations，Helm hooks 也是 annotations。Stable selector labels 必須一致，checksum 或 release notes 等變動資料不得進入 selectors。

建議 labels：

```yaml
metadata:
  labels:
    app.kubernetes.io/name: {{ include "mychart.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: example-platform
```

## 建立安全的 Pod templates

- 使用固定 image tag；immutable promotion 則使用 digest。Released workload 不使用 `latest`、`head` 或 `canary` 等 floating tags。
- Repository、tag/digest 與 pull policy 都由 documented values 提供；non-`latest` image 通常明確預設 `IfNotPresent`。
- 明確宣告 `spec.selector.matchLabels`，並與 `spec.template.metadata.labels` 的 stable labels 完全一致；version 或 release date 不放入 selector。
- 依 workload 納入 resource requests/limits、health probes、security contexts、graceful termination、topology/availability rules 與 disruption protection。

## 將 CRD 視為獨立 lifecycle

- CRD declarations 放在 chart 的 `crds/`，Helm 在其他 resources 前安裝這些 non-templated files，已存在時跳過。
- 區分 CRD declaration 與使用該 API 的 custom resources。
- Helm normal lifecycle 不會 upgrade 或 delete CRDs，以避免 data loss；必須另訂 reviewed CRD upgrade 與 compatibility procedure。
- `helm install --dry-run` 與 `helm upgrade --dry-run` 無法一起驗證新 CRD 及其 custom resources；使用 `helm template`、獨立 CRD chart 或 ephemeral test cluster。
- CRDs 被多個 charts 共用、需要獨立 ownership，或必須先升級時，考慮 separate CRD chart。

## 讓 RBAC 可設定且 least-privileged

- `rbac` 與 `serviceAccount` 使用不同 values keys。
- `rbac.create` 預設 `true`，但允許 platform team 關閉 chart 對 RBAC resources 的 ownership。
- 支援 `serviceAccount.create: false` 與 `serviceAccount.name` 引用既有 ServiceAccount；需建立且未給名稱時由 fullname helper 產生。
- 每個 Pod template 都綁定選定的 ServiceAccount。
- 優先使用 namespaced Roles/RoleBindings；只有真正需要 cluster scope 才使用 ClusterRoles/ClusterRoleBindings，不授予沒有理由的 wildcard permissions。
- 不呼叫 Kubernetes API 的 workloads 設定 `automountServiceAccountToken: false`。

```yaml
rbac:
  # rbac.create specifies whether the chart creates RBAC resources.
  create: true

serviceAccount:
  # serviceAccount.create specifies whether the chart creates a ServiceAccount.
  create: true
  # serviceAccount.name is the ServiceAccount to use; empty generates a name.
  name: ""
  # serviceAccount.automount specifies whether to mount an API token.
  automount: false
```

## Release 前驗證

針對每組 supported values 與 Kubernetes version 執行：

```console
helm dependency build ./chart
helm lint ./chart --strict
helm template test ./chart --namespace test --values values-test.yaml > rendered.yaml
kubectl apply --dry-run=server --filename rendered.yaml
helm upgrade --install test ./chart --namespace test --create-namespace \
  --atomic --wait --timeout 5m
helm test test --namespace test
```

依風險加入 schema checks、policy validation、API deprecation checks 與 ephemeral-cluster install。Render 成功只證明 template syntax，不證明 rollout health 或 application behavior。

## Review checklist

- [ ] Chart names、versions、YAML 與 namespace handling 符合 conventions。
- [ ] 每個 value 都有文件、明確 type 且容易 override。
- [ ] Secret 不會進入 Git、command history、release metadata 或 rendered output。
- [ ] Template files、helper names、whitespace 與 comments 一致。
- [ ] Dependencies 有 reviewed constraints、conditions 與 committed lock file。
- [ ] Recommended labels 完整，selectors 只使用 stable labels。
- [ ] Images immutable，Pod templates 包含必要 operational/security controls。
- [ ] CRD installation、upgrade、compatibility 與 ownership 明確。
- [ ] RBAC 與 ServiceAccount creation 可設定且 least-privileged。
- [ ] CI 會 lint、render、validate、install、等待 readiness 並以 representative values 測試。

## 參考資料

- [General conventions](https://helm.sh/docs/chart_best_practices/conventions/)
- [Values](https://helm.sh/docs/chart_best_practices/values/)
- [Templates](https://helm.sh/docs/chart_best_practices/templates/)
- [Dependencies](https://helm.sh/docs/chart_best_practices/dependencies/)
- [Labels and annotations](https://helm.sh/docs/chart_best_practices/labels/)
- [Pods and PodTemplates](https://helm.sh/docs/chart_best_practices/pods/)
- [Custom Resource Definitions](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)
- [Role-Based Access Control](https://helm.sh/docs/chart_best_practices/rbac/)

---

# Helm Chart Best Practices

Use this guide to build charts that are predictable to configure, easy to
operate, and safe to upgrade. Treat chart templates, default values, dependency
metadata, and rendered manifests as production code.

## Follow chart conventions

- Name charts with DNS-1123 labels: lowercase letters, numbers, and dashes,
  starting and ending with a letter or number, with at most 63 characters.
- Use Semantic Versioning for chart versions. When placing a version in a
  Kubernetes label, replace `+` with `_` because label values cannot contain
  `+`.
- Indent YAML with two spaces and never use tabs.
- Use `Helm` for the project, `helm` for the command, `chart` as a common noun,
  and the case-sensitive filename `Chart.yaml`.
- Do not hard-code `metadata.namespace` in namespaced templates. Select the
  target namespace at deployment time with `--namespace` or the equivalent
  GitOps configuration.

## Design values for users

- Start user-defined value names with a lowercase letter and use camel case,
  such as `servicePort`.
- Prefer flat values when they keep templates simpler. Use nested maps when
  several related settings form a clear group, especially when at least one is
  required.
- Prefer maps over positional lists when users may override individual entries
  with `--set`; map keys remain stable when entries are reordered.
- Quote strings. For large numeric identifiers that must survive YAML parsing
  unchanged, store them as strings and convert them explicitly in templates.
- Document every property in `values.yaml`. Begin each comment with the exact
  property name so users and documentation tools can find it reliably.
- Keep secrets out of committed values, command-line `--set` arguments, release
  metadata, CI logs, and rendered manifests. Reference an externally managed
  Secret instead.
- Use `values.schema.json` when practical to reject invalid types, ranges, and
  required-value omissions before installation.

```yaml
# replicaCount is the desired number of application replicas.
replicaCount: 2

# image configures the application container image.
image:
  repository: "registry.example.com/team/service"
  tag: "1.4.2"
  pullPolicy: IfNotPresent

# service configures the Kubernetes Service.
service:
  port: 8080
```

## Keep templates maintainable

- Use `.yaml` for templates that render YAML and `.tpl` for helper-only files.
- Use dashed filenames and include the resource kind, such as
  `api-deployment.yaml`. Put each resource definition in its own file.
- Namespace every defined template because helpers are global across a chart
  and all subcharts, for example `{{ define "mychart.fullname" }}`.
- Use two-space indentation and spaces inside template delimiters. Chomp
  whitespace deliberately and inspect the rendered result for excess blank
  lines or malformed YAML.
- Use Helm template comments (`{{/* ... */}}`) for implementation notes that
  should not appear in rendered output. Use YAML comments only when operators
  should see them in rendered manifests.
- Use JSON syntax inside YAML only where it improves a small construct's
  readability; keep complex structures in YAML.
- Centralize names, selectors, and standard labels in `_helpers.tpl` to prevent
  drift between resources.
- Fail clearly on mandatory inputs with schema validation or `required`, and
  apply `quote`, `toYaml`, `nindent`, and type conversions intentionally.

```yaml
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## Manage dependencies explicitly

- Declare dependencies in `Chart.yaml` and commit the generated `Chart.lock` so
  CI and developers resolve the same dependency set.
- Use an appropriate Semantic Versioning constraint. The Helm guide suggests a
  patch-level range such as `~1.2.3`; use an exact version instead when your
  release policy requires fully controlled updates.
- Prefer HTTPS repositories. Use `file://` only for charts assembled by a fixed
  pipeline, and remember that downloader-plugin URLs require users to install
  the same plugin.
- Give every optional dependency a condition such as `database.enabled`.
  Shared tags are useful when several subcharts together implement one optional
  feature.
- Run `helm dependency update` deliberately when changing constraints, then
  review both `Chart.lock` and the downloaded dependency versions.

The referenced Helm dependency page states that it has not yet been updated for
Helm 4. Confirm its version-range and dependency behavior against the Helm
version used by the delivery pipeline.

## Apply consistent metadata

Use labels for metadata that Kubernetes or operators query. Use annotations for
non-identifying metadata; Helm hooks are annotations. Apply stable selector
labels consistently and keep changing data, such as checksums or release notes,
out of selectors.

Recommended labels include:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: {{ include "mychart.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: example-platform
```

## Build safe Pod templates

- Use a fixed image tag or, for immutable promotion, an image digest. Never use
  floating tags such as `latest`, `head`, or `canary` for released workloads.
- Expose the repository, tag or digest, and pull policy through documented
  values. Default `imagePullPolicy` deliberately; `IfNotPresent` is the normal
  default for non-`latest` images.
- Declare `spec.selector.matchLabels` explicitly and make it exactly match the
  stable labels in `spec.template.metadata.labels`. Do not include version or
  release-date labels in selectors.
- Include production controls appropriate to the workload: resource requests
  and limits, health probes, security contexts, graceful termination,
  topology/availability rules, and disruption protection.

## Handle CRDs as a separate lifecycle

- Put CRD declarations in the chart's `crds/` directory. Helm installs these
  non-templated files before other resources and skips CRDs that already exist.
- Distinguish CRD declarations from custom resources that use those APIs.
- Helm does not upgrade or delete CRDs during normal chart lifecycle operations
  because doing so can cause data loss. Define an explicit, reviewed CRD upgrade
  and compatibility procedure.
- `helm install --dry-run` and `helm upgrade --dry-run` cannot validate a new CRD
  and its custom resources together. Use `helm template`, a separate CRD chart,
  or an ephemeral test cluster for validation.
- Consider a separate CRD chart when CRDs are shared, need independent release
  ownership, or must be installed and upgraded before application charts.

## Make RBAC configurable and least-privileged

- Keep `rbac` and `serviceAccount` under separate values keys.
- Default `rbac.create` to `true`, while allowing platform teams to disable chart
  ownership of RBAC resources.
- Support an existing ServiceAccount with `serviceAccount.create: false` and
  `serviceAccount.name`. If creation is enabled and no name is supplied,
  generate one with the chart's fullname helper.
- Bind workloads to the selected ServiceAccount in every Pod template.
- Prefer namespaced Roles and RoleBindings. Use ClusterRoles or
  ClusterRoleBindings only for capabilities that genuinely require cluster
  scope, and never grant wildcard permissions without a documented reason.
- Set `automountServiceAccountToken: false` for workloads that do not call the
  Kubernetes API.

```yaml
rbac:
  # rbac.create specifies whether the chart creates RBAC resources.
  create: true

serviceAccount:
  # serviceAccount.create specifies whether the chart creates a ServiceAccount.
  create: true
  # serviceAccount.name is the ServiceAccount to use; empty generates a name.
  name: ""
  # serviceAccount.automount specifies whether to mount an API token.
  automount: false
```

## Validate before release

Run validation against every supported values set and Kubernetes version:

```console
helm dependency build ./chart
helm lint ./chart --strict
helm template test ./chart --namespace test --values values-test.yaml > rendered.yaml
kubectl apply --dry-run=server --filename rendered.yaml
helm upgrade --install test ./chart --namespace test --create-namespace \
  --atomic --wait --timeout 5m
helm test test --namespace test
```

Add schema checks, policy validation, API deprecation checks, and an ephemeral
cluster install when the chart's risk warrants them. A successful render proves
template syntax, not rollout health or application behavior.

## Review checklist

- [ ] Chart names, versions, YAML, and namespace handling follow conventions.
- [ ] Every value is documented, typed clearly, and practical to override.
- [ ] No secret can enter Git, command history, release metadata, or rendered
      output.
- [ ] Template files, helper names, whitespace, and comments are consistent.
- [ ] Dependencies have reviewed constraints, conditions, and a committed lock
      file.
- [ ] Recommended labels are present and selectors use only stable labels.
- [ ] Images are immutable and Pod templates include required operational and
      security controls.
- [ ] CRD installation, upgrades, compatibility, and ownership are explicit.
- [ ] RBAC and ServiceAccount creation are configurable and least-privileged.
- [ ] CI lints, renders, validates, installs, waits for readiness, and tests the
      chart with representative values.

## References

- [General conventions](https://helm.sh/docs/chart_best_practices/conventions/)
- [Values](https://helm.sh/docs/chart_best_practices/values/)
- [Templates](https://helm.sh/docs/chart_best_practices/templates/)
- [Dependencies](https://helm.sh/docs/chart_best_practices/dependencies/)
- [Labels and annotations](https://helm.sh/docs/chart_best_practices/labels/)
- [Pods and PodTemplates](https://helm.sh/docs/chart_best_practices/pods/)
- [Custom Resource Definitions](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)
- [Role-Based Access Control](https://helm.sh/docs/chart_best_practices/rbac/)
