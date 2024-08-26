### To debug with [go-delve](https://github.com/go-delve/delve)

```makefile
# Run with Delve for development purposes against the configured Kubernetes cluster in ~/.kube/config
# Delve is a debugger for the Go programming language. More info: https://github.com/go-delve/delve
run-delve: generate fmt vet manifests
    go build -gcflags "all=-trimpath=$(shell go env GOPATH)" -o bin/manager main.go
    dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./bin/manager
```

### To change the version of CRDs

`controller-gen` 允许声明生成 CRD API 的版本，通过 `CRD_OPTIONS` 控制。

```makefile
CRD_OPTIONS ?= "crd:crdVersions={v1beta1},preserveUnknownFields=false"

manifests: controller-gen
    $(CONTROLLER_GEN) rbac:roleName=manager-role $(CRD_OPTIONS) webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

### To get all the manifests without deploying

++ dry-run

```makefile
dry-run: manifests
    cd config/manager && $(KUSTOMIZE) edit set image controller=${IMG}
    mkdir -p dry-run
    $(KUSTOMIZE) build config/default > dry-run/manifests.yaml
```

