To modify a scaffolded proj by `go.mod` for APIs & Controllers.

:smile: Separate `go.mod` modules

- 企业版 Operator 想要复用社区版的 API 时
- 许多外部 modules 依赖 API，想要严格区分过渡依赖
- 当 API 引入到其他项目时减少过渡依赖的影响
- 将 API 的发布过程与 Controller 的发布过程分开管理。
- 模块化代码库，但不想将代码分割到多个存储库中

:cry: 但这也带来了一些问题，使它难以成为通用化和插件化

- 多 `go.mod` 不是最佳实践
- 总是可以将 API 提取到一个新的存储库中，并在跨多个依赖相同 API 类型的存储库的项目中，更好地控制发布过程。
- 至少需要 1 个 replace directive
  - 通过 go.work + 至少 2 个文件 & 环境变量 GO_WORK
  - go.mod replace 必须在每一次发布进行添加/删除

### Adujusting your Project

```bash
$ kubebuilder init
$ kubebuilder create api \
	--group operator \
	--version v1alpha1 \
	--kind Sample \
	--resource \
	--controller \
	--make
```

`go.mod` 只包含 apimachinery & controller-runtime，任何在 controller 中的依赖不会被包含。

```bash
$ cd api/v1alpha1
$ go mod init
$ go mod tidy
```

### Using replace directives for development

当在 operator root folder 进行 go mod tidy 时，会报错...

原因：尚未 push module 到 VCS，导致无法解析

```bash
go: finding module for package YOUR_GO_PATH/test-operator/api/v1alpha1
YOUR_GO_PATH/test-operator imports
    YOUR_GO_PATH/test-operator/api/v1alpha1: cannot find module providing package YOUR_GO_PATH/test-operator/api/v1alpha1: module YOUR_GO_PATH/test-operator/api/v1alpha1: git ls-remote -q origin in LOCALVCSPATH: exit status 128:
    remote: Repository not found.
    fatal: repository 'https://YOUR_GO_PATH/test-operator/' not found
```

解决：

1. 使用 `go.mod`

```bash
# Only if you didn't already resolve the module
go mod edit -require YOUR_GO_PATH/test-operator/api/v1alpha1@v0.0.0
go mod edit -replace YOUR_GO_PATH/test-operator/api/v1alpha1@v0.0.0=./api/v1alpha1
# Since the main go.mod file now has a replace directive, it is important to drop it again
go mod edit -dropreplace YOUR_GO_PATH/test-operator/api/v1alpha1
go mod tidy
```

2. 使用 `go.workspaces`

```bash
$ go work init
$ go work use .             # include main module
$ go work use api/v1alpha1  # include API sub module
$ go work sync

# ++ in .gitignore
go.work
go.work.sum
```

### Adjusting the Dockerfile

```docker
# Build the manager binary
FROM golang:1.20 as builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# Copy the Go Sub-Module manifests
COPY api/v1alpha1/go.mod api/go.mod
COPY api/v1alpha1/go.sum api/go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
RUN go mod download

# Copy the go source
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/controller/ internal/controller/

# Build
# the GOARCH has not a default value to allow the binary be built according to the host where the command
# was called. For example, if we call make docker-build in a local env which has the Apple Silicon M1 SO
# the docker BUILDPLATFORM arg will be linux/arm64 when for Apple x86 it will be linux/amd64. Therefore,
# by leaving it empty we can ensure that the container and binary shipped on it will have the same platform.
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager cmd/main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]
```

### Creating a new API and controller release

After this, your modules will be available in VCS and you do not need a local replacement anymore.

若本地做了修改，确保对应的 replace directives 生效。

```bash
$ git commit
$ git tag v1.0.0 # this is your main module release
$ git tag api/v1.0.0 # this is your api release
$ go mod edit -require YOUR_GO_PATH/test-operator/api@v1.0.0 # now we depend on the api module in the main module
$ go mod edit -dropreplace YOUR_GO_PATH/test-operator/api/v1alpha1 # this will drop the replace directive for local development in case you use go modules, meaning the sources from the VCS will be used instead of the ones in your monorepo checked out locally.
$ git push origin main v1.0.0 api/v1.0.0

```

### Reuse

```bash
$ go get YOUR_GO_PATH/test-operator/api@v1.0.0
```

