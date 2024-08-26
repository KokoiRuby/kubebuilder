编写 Controller 可引用多个不同的 external types。**kubebuilder 只支持 1/2，3/4 需要手动 scaffold**

1. **CRD in current proj**
2. **Core K8s Resources**
3. CRD from another proj
4. Custom API defined in aggregation layer, served by ext. API Server

为了使用另一个 proj 中的 CRD，需要以下信息

- CR Domain
- Group under Domain
- The Go import path of the CR Type definition
- CR Type

**Example**: `github.com/theiruser/theirproject/apis/theirgroup/v1alpha1` where

- The Domain of the CR is `theirs.com`
- Group is `theirgroup`
- Kind/Go type `ExternalTypeKind`

### Tips

```bash
$ kubectl api-resources --verbs=list -o name
$ kubectl api-resources --verbs=list -o name | grep my.domain
```

### Prerequisites

未指定 `--domain` 默认 my.domain

```bash
$ kubebuilder init
```

### Add a controller for the external Type

`--resource=false` 不创建 CRD，选择依赖外部类型。

```bash
$ kubebuilder create api \
	--group theirgroup \
	--version v1alpha1 \
	--kind ExternalTypeKind \
	--controller \
	--resource=false
```

`./PROJECT`

```yaml
domain: my.domain
layout:
- go.kubebuilder.io/v4
projectName: ext-type
repo: std
resources:
- controller: true
  # domain: my.domain
  domain: theirs.com   # replace to ext domain
  group: theirgroup
  kind: ExternalTypeKind
  version: v1alpha1
version: "3"
```

RBAC `./internal/controller/externaltype_controller.go`

```go
// external types can be added like this
// +kubebuilder:rbac:groups=theirgroup.theirs.com,resources=externaltypes,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=theirgroup.theirs.com,resources=externaltypes/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=theirgroup.theirs.com,resources=externaltypes/finalizers,verbs=update

// core types can be added like this
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods/status,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods/finalizers,verbs=update
```

### Register your (ext.) Types

`./cmd/main.go`

```go
import (
    theirgroupv1alpha1 "github.com/theiruser/theirproject/apis/theirgroup/v1alpha1"  // ++
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(theirgroupv1alpha1.AddToScheme(scheme)) // ++
    // +kubebuilder:scaffold:scheme
}
```

### Use the correct imports for your API and uncomment the controlled resource

`./internal/controllers/externaltype_controller.go`

```go
package controllers

import (
    theirgroupv1alpha1 "github.com/theiruser/theirproject/apis/theirgroup/v1alpha1"
)

// SetupWithManager sets up the controller with the Manager.
func (r *ExternalTypeReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&theirgroupv1alpha1.ExternalType{}).  // ++
        Complete(r)
}
```

### Update dependencies

```bash
$ go mod tidy
```

### Generate RBAC

```bash
$ make manifests
```

