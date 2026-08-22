# Dockerfile 最佳實務

本指南用來建立安全、可重現、體積小且建置快速的 container images。Dockerfile、build context 與 dependency lock files 都應視為 production source code。

## 從可信任的 minimal image 開始

- 優先使用 Docker Official Images、Verified Publisher images 或內部核准的 base image。
- 選擇仍能滿足 libraries 與 debugging 需求的最小 base；minimal image 降低 transfer 與 attack surface，但熟悉的 slim distribution 可能比沒有 shell 的 image 更容易操作。
- Production base image pin 到 immutable digest；tag 可讀但會移動，digest 才能重現內容。
- 定期 rebuild，讓 patched base 與 dependencies 進入 production；digest pins 也要由 automation 或例行維護更新。

```Dockerfile
FROM python:3.13.7-slim@sha256:<verified-digest>
```

## 使用 multi-stage builds

將 compilers、package managers、source、test data 與 build-only content 排除於 runtime image。每個 stage 命名，final stage 只 copy 必要 artifacts。

```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.26 AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -o /out/service ./cmd/service

FROM gcr.io/distroless/static-debian13:nonroot
COPY --from=build /out/service /service
USER nonroot:nonroot
ENTRYPOINT ["/service"]
```

CI 必須 build 並測試真正的 production stage；builder stage 成功不代表 final image 能啟動。

## 以 non-root user 執行

Containers 預設以 root 執行。Container root 不必然等同 host root，但會放大 container escape、過度 Linux capabilities 或 writable host mount 的影響。

- 切換 user 前安裝 OS packages；image 不安裝 `sudo`。
- Base image 沒有專用 user 時，以明確 UID/GID 建立。
- 在 `COPY` 時設定 ownership，避免新增 recursive `chown` layer。
- Application 應能使用 read-only root filesystem，只寫入 `/tmp`、`/data` 等明確 runtime volumes。
- Kubernetes security context 再設定 `runAsNonRoot`、`allowPrivilegeEscalation: false`、drop capabilities 與 read-only root filesystem。Dockerfile 無法單獨強制 runtime controls。

Debian-based example：

```Dockerfile
ARG UID=10001
ARG GID=10001

RUN groupadd --gid "${GID}" app \
    && useradd --uid "${UID}" --gid "${GID}" --no-create-home \
        --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app ./app ./app
USER 10001:10001
```

Alpine 的 `addgroup`/`adduser` flags 與 Debian tools 不同，必須依選定版本確認。

## 只安裝 runtime 需要的內容

- 使用 application lock file 與 deterministic package-manager mode。
- 不安裝非必要的 recommended/suggested OS packages。
- 在同一個 `RUN` 完成 repository metadata refresh、package installation 與 cleanup；不得重用較早 layer 的 stale package indexes。
- 長 package list 排序，方便發現重複與意外變更。
- Package cache 只從 image layer 移除；BuildKit cache mount 可把下載 cache 保留在 final image 外。

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

## 保護 build secrets

Credentials 絕不放入 `ARG`、`ENV`、copied config 或 registry URL；它們可能留在 metadata、history、layers、logs 或 cache。使用 BuildKit secret/SSH mounts，只在需要的 instruction 中 consume。

```Dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```console
docker build --secret id=npmrc,src="$HOME/.npmrc" .
```

並以 `.dockerignore` 排除 local credentials、environment files、VCS metadata、test output 與 build artifacts。

```gitignore
.git
.env*
node_modules
coverage
dist
```

## 最佳化 build cache

Instructions 從穩定到常變排序。先 copy dependency manifests 並 install dependencies，再 copy application source，避免 source edit 使 dependency layer 失效。

```Dockerfile
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build
```

Build context 保持精簡。Ephemeral CI builder 可在 platform 與 registry trust model 允許時用 `--cache-to`/`--cache-from` 匯出與匯入 BuildKit cache。

## 有意識地使用 Dockerfile instructions

- `RUN` 執行 build-time command 並產生 filesystem layer。相關 installation/cleanup 可合併，不相關工作則保持足夠分離以利 readability 與 cache。
- `CMD` 定義可被覆寫的 default command，或 exec-form `ENTRYPOINT` 的 default arguments；`docker run` image name 後的 arguments 會取代 `CMD`。
- `ENTRYPOINT` 定義 primary executable；runtime arguments 會附加到 exec-form `ENTRYPOINT`，也可用 `docker run --entrypoint` 明確取代。
- Build context 檔案優先用 `COPY`；只有確實需要 local archive auto-extraction 等額外行為才用 `ADD`。
- `WORKDIR` 使用 absolute path，不依賴 base image current directory。
- 除非需要初始化，不使用 wrapper script；需要時最後以 `exec "$@"` 讓 application 取代 shell process。
- `ENV` 只放安全 runtime defaults，不 bake environment-specific config 或 secrets。
- `EXPOSE` 只是文件，不會 publish port 或限制 network access。
- 加入 source repository、revision 等有用 OCI labels。

```Dockerfile
WORKDIR /app
COPY --chown=10001:10001 service /app/service

USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/app/service"]
CMD ["--listen=:8080"]
```

需要 variable expansion、pipeline、redirection 或 chaining 的 `RUN` 適合 shell form：

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

`ENTRYPOINT` 與 `CMD` 優先用 exec form。它不呼叫 `/bin/sh -c`，application 會成為 container PID 1 並直接收到 termination signals；application 仍須處理 signals 與 reap child processes，必要時使用 minimal init。Exec form 不做 shell expansion；只有確實需要才明確叫 shell。

Kubernetes workloads 的 liveness、readiness、startup probes 放在 Pod spec，因 thresholds 是 deployment concern。若 image 也需為 plain Docker 或其他 runtime 提供 default，才加入 Dockerfile `HEALTHCHECK`，並確保 probe binary 存在。

## 安全地驗證與發布

CI 應：

1. Lint Dockerfile 並以 BuildKit build。
2. 執行 unit tests，啟動 final image 做 smoke test。
3. 掃描 Dockerfile、dependencies 與 final image 的 vulnerabilities、malware、exposed secrets 與 policy violations。
4. 產生 SBOM，與 image provenance 一起保存。
5. Image 只 push 一次，以 digest deployment，environments 間 promote 同一 digest，不重建。
6. Delivery platform 會驗證 signature 時簽署 image。

乾淨的 vulnerability scan 不會永久有效；advisory database 更新後要重新掃描 published images，修補方式是 rebuild，而非 patch running containers。

## Review checklist

- [ ] Base image 可信任、minimal、受支援且 production 已 pin。
- [ ] Named stages 分隔 build 與 runtime content。
- [ ] Final image 沒有 compiler、package cache、source、test data 或 credentials。
- [ ] Runtime user 使用固定 non-zero UID 且只有必要 write access。
- [ ] Dependencies locked 且 deterministic install。
- [ ] `.dockerignore` 排除無關與敏感檔案。
- [ ] Build secrets 使用 BuildKit secret/SSH mounts。
- [ ] Cache-friendly layers 先 copy dependency manifests。
- [ ] `ENTRYPOINT`/`CMD` 使用 exec form，process 能處理 termination。
- [ ] CI build、test、scan、inventory 並只發布 final image 一次。
- [ ] Deployment 引用 promoted image digest。

## 參考資料

- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Choosing between RUN, CMD, and ENTRYPOINT](https://www.docker.com/blog/docker-best-practices-choosing-between-run-cmd-and-entrypoint/)
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Optimize build cache usage](https://docs.docker.com/build/cache/optimize/)
- [Build secrets](https://docs.docker.com/build/building/secrets/)

---

# Dockerfile Best Practices

Use this guide to produce container images that are secure, reproducible, small,
and quick to build. Treat the Dockerfile, its build context, and dependency lock
files as production source code.

## Start from a trusted, minimal image

- Prefer Docker Official Images, Verified Publisher images, or an internally
  approved base image.
- Choose the smallest base that still provides the libraries and debugging
  characteristics the application needs. A minimal image reduces transfer time
  and attack surface, but a familiar slim distribution may be easier to operate
  than a shell-less image.
- Pin production base images to an immutable digest. A tag is readable but can
  move; a digest makes the selected content reproducible.
- Rebuild regularly so that patched base images and dependencies reach
  production. Digest pins must be updated deliberately by automation or routine
  maintenance.

```Dockerfile
FROM python:3.13.7-slim@sha256:<verified-digest>
```

## Use multi-stage builds

Keep compilers, package managers, source files, test data, and other build-only
content out of the runtime image. Name each stage and copy only the required
artifacts into the final stage.

```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.26 AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -o /out/service ./cmd/service

FROM gcr.io/distroless/static-debian13:nonroot
COPY --from=build /out/service /service
USER nonroot:nonroot
ENTRYPOINT ["/service"]
```

Build and test the exact production stage in CI. Do not assume that a successful
builder stage proves the final image starts correctly.

## Run as a non-root user

Containers run as root by default. Container root is not automatically host
root, but it increases the impact of a container escape, excessive Linux
capabilities, or a writable host mount.

- Install operating-system packages before switching users; do not install
  `sudo` in the image.
- Create a dedicated user with an explicit UID and GID when the base image does
  not already provide one.
- Set ownership during `COPY` instead of adding a separate recursive `chown`
  layer.
- Ensure the application can run with a read-only root filesystem and write only
  to explicit runtime volumes such as `/tmp` or `/data`.
- Set `runAsNonRoot`, `allowPrivilegeEscalation: false`, dropped capabilities,
  and a read-only root filesystem in the Kubernetes security context as
  defense in depth. The Dockerfile alone cannot enforce those runtime controls.

Debian-based example:

```Dockerfile
ARG UID=10001
ARG GID=10001

RUN groupadd --gid "${GID}" app \
    && useradd --uid "${UID}" --gid "${GID}" --no-create-home \
        --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app ./app ./app
USER 10001:10001
```

Use `addgroup` and `adduser` with the flags supported by the selected Alpine
version when building from Alpine; their options differ from Debian's tools.

## Install only what the runtime needs

- Use the application's lock file and deterministic package-manager mode.
- Do not install recommended or suggested operating-system packages unless they
  are required.
- Combine repository metadata refresh, package installation, and metadata
  cleanup in one `RUN` instruction. Never reuse stale package indexes from an
  earlier layer.
- Sort long package lists to make duplicate and accidental changes visible.
- Remove package caches only from the image layer. BuildKit cache mounts can
  retain download caches outside the final image.

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

## Protect secrets during builds

Never place credentials in `ARG`, `ENV`, copied configuration files, or registry
URLs. They can remain in image metadata, build history, layers, logs, or cache.
Use BuildKit secret or SSH mounts and consume the secret only in the instruction
that needs it.

```Dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```console
docker build --secret id=npmrc,src="$HOME/.npmrc" .
```

Also exclude local credentials, environment files, VCS metadata, test output,
and build artifacts with `.dockerignore`.

```gitignore
.git
.env*
node_modules
coverage
dist
```

## Optimize build caching

Order instructions from stable to frequently changed. Copy dependency manifests
and install dependencies before copying application source so that source edits
do not invalidate the dependency layer.

```Dockerfile
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build
```

Keep the build context small. For ephemeral CI builders, export and import a
BuildKit cache with `--cache-to` and `--cache-from` when the CI platform and
registry trust model permit it.

## Use Dockerfile instructions deliberately

- Use `RUN` to execute build-time commands that configure the image. Each
  `RUN` instruction produces a filesystem layer, so combine related package
  installation and cleanup steps, while keeping unrelated work separate enough
  to preserve readable output and useful build caching.
- Use `CMD` to define an overridable default command, or default arguments for
  an exec-form `ENTRYPOINT`. Arguments supplied after the image name in
  `docker run` replace `CMD`.
- Use `ENTRYPOINT` for the image's primary executable. Runtime arguments are
  appended to an exec-form `ENTRYPOINT`; users can still replace it explicitly
  with `docker run --entrypoint`.
- Prefer `COPY` for files from the build context. Use `ADD` only when its extra
  behavior, such as automatic local archive extraction, is intentional.
- Use absolute paths with `WORKDIR`; do not rely on the base image's current
  directory.
- Avoid wrapper scripts unless they perform necessary initialization and end
  with `exec "$@"` so the application replaces the shell process.
- Use `ENV` only for safe runtime defaults. Do not bake environment-specific
  configuration or secrets into the image.
- Treat `EXPOSE` as documentation; it neither publishes the port nor restricts
  network access.
- Add OCI labels for useful metadata such as source repository and revision.

```Dockerfile
WORKDIR /app
COPY --chown=10001:10001 service /app/service

USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/app/service"]
CMD ["--listen=:8080"]
```

Prefer shell form for a `RUN` instruction that needs shell features such as
variable expansion, pipelines, redirection, or command chaining:

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

Prefer exec form for `ENTRYPOINT` and `CMD`. It does not invoke `/bin/sh -c`, so
the application becomes the container's PID 1 and receives termination signals
directly. The application must still handle those signals and reap child
processes, or the image should use a minimal init process when needed. Exec form
does not provide shell expansion; invoke a shell explicitly only when that
behavior is required.

For Kubernetes workloads, define liveness, readiness, and startup probes in the
Pod specification because their thresholds are deployment concerns. Add a
Dockerfile `HEALTHCHECK` when the image must also provide a useful default for
plain Docker or another runtime, and ensure the required probe binary exists.

## Validate and publish safely

CI should:

1. Lint the Dockerfile and build it with BuildKit.
2. Run unit tests and start the final image for a smoke test.
3. Scan the Dockerfile, dependencies, and final image for vulnerabilities,
   malware, exposed secrets, and policy violations.
4. Generate an SBOM and retain it with the image provenance.
5. Push the image once, deploy by digest, and promote the same digest between
   environments instead of rebuilding it.
6. Sign the image when the delivery platform verifies signatures.

Do not treat a clean vulnerability scan as permanent. Re-scan published images
as advisory databases change, and rebuild rather than patching running
containers.

## Review checklist

- [ ] The base image is trusted, minimal, supported, and pinned for production.
- [ ] Build and runtime content are separated with named stages.
- [ ] The final image contains no compiler, package cache, source, test data, or
      credentials.
- [ ] The runtime user has a fixed non-zero UID and only required write access.
- [ ] Dependencies are locked and installed deterministically.
- [ ] `.dockerignore` excludes irrelevant and sensitive files.
- [ ] Build secrets use BuildKit secret or SSH mounts.
- [ ] Cache-friendly layers copy dependency manifests before application source.
- [ ] `ENTRYPOINT` and `CMD` use exec form and the process handles termination.
- [ ] CI builds, tests, scans, inventories, and publishes the final image once.
- [ ] Deployments reference the promoted image digest.

## References

- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Choosing between RUN, CMD, and ENTRYPOINT](https://www.docker.com/blog/docker-best-practices-choosing-between-run-cmd-and-entrypoint/)
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Optimize build cache usage](https://docs.docker.com/build/cache/optimize/)
- [Build secrets](https://docs.docker.com/build/building/secrets/)
