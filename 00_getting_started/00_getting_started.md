### [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

使用 Kubernetes API 和 Custom Controller 对 Custom Resource 进行 Reconcile。

### Sample Project

- Reconcile Memcached CR 表示一个 Memcached deployment 实例
- 使用 Memcahced image 创建 Memcached deployment
- 不允许创建超过 CR 中描述指定副本数实例
- 更新 Memcached CR status

### [Create a project](https://book.kubebuilder.io/getting-started#create-a-project)

```bash
# $GOPATH/src/github.com/kubebuilder/00_getting_started/
$ mkdir memcached-operator && cd memcached-operator
$ kubebuilder init --domain=example.com
```

### [Create the Memcached API (CRD)](https://book.kubebuilder.io/getting-started#create-the-memcached-api-crd)

`--plugins` 指定要使用的 Kubebuilder 插件

`--make` 指定是否生成 Makefile 文件

```bash
$ kubebuilder create api \
    --group cache \
    --version v1alpha1 \
    --kind Memcached \
    --image=memcached:1.4.36-alpine \
    --image-container-command="memcached,-m=64,-o,modern,-v" \
    --image-container-port="11211" \
    --run-as-user="1001" \
    --plugins="deploy-image/v1-alpha" \
    --make=false
```

GVK uniquely **identifies** the new CRD.

`api/v1alpha1/memcached_types.go`

- **Spec**: avail conf for CR.
- **Status**: observation of CR.

Utilize [controller-gen](https://book.kubebuilder.io/reference/controller-gen) to generate the CRD manifest under `config/crd/bases`, while CR sample is under `config/samples`.

```bash
$ make generate
```

[Markers](https://book.kubebuilder.io/reference/markers) to help in **validations & criteria**.

```go
// api/v1alpha1/memcached_types.go
type MemcachedSpec struct {
    ...
    // +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=3
	Size int32 `json:"size,omitempty"`
    ...
}
```

```yaml
# config/crd/bases/cache.example.com_memcacheds.yaml
spec:
  description: MemcachedSpec defines the desired state of Memcached
  properties:
    ...
    size:
      description: |
        Size defines the number of Memcached instances.
        The following markers will use OpenAPI v3 schema to validate the value.
        More info: https://book.kubebuilder.io/reference/markers/crd-validation.html
      format: int32
      maximum: 3
      minimum: 1
      type: integer
  type: object

```

### Reconciliation Process

核心**业务逻辑**确保 resource 和 spec 的同步性，以循环的形式，持续检查 conditions 并执行对应的逻辑。

`internal/controller/memcached_controller.go`

1. 获取 CR 实例
2. 若无状态，设置成 Unknown
3. 添加 finalizer
4. 检查 CR 实例是否被删除，即查看 deletion timestamp；若删除，置状态 Downgrade 并执行 finalizer
5. 检查是否已创建 low-level 资源；若无则创建。
6. 检查约束的副本数是否符合 spec 要求。

```go
// pseudo-code
reconcile App {

  // Check if a Deployment/Service for the app exists, if not, create one
  // If there's an error, then restart from the beginning of the reconcile
  if err != nil {
    return reconcile.Result{}, err
  }

  // Look for Database CR/CRD
  // Check the Database Deployment's replicas size
  // If deployment.replicas size doesn't match cr.size, then update it
  // Then, restart from the beginning of the reconcile.
  if err != nil {
    return reconcile.Result{Requeue: true}, nil
  }
  ...

  // If at the end of the loop:
  // Everything was executed successfully, and the reconcile can stop
  return reconcile.Result{}, nil

}

```

```go
// possible return
return ctrl.Result{}, err                                   // with err
return ctrl.Result{Requeue: true}, nil                      // without err
return ctrl.Result{}, nil                                   // stop the Reconcile
return ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())}, nil // reconcile again after X times
```

Controller 持续观察，监控任何关联 Kind 的各种事件 create/update/delete

kubebuilder 生成代码中已经实现了 [watch](https://book.kubebuilder.io/reference/watching-resources)。

```go
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&cachev1alpha1.Memcached{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}
```

**RBAC**

[markers](https://book.kubebuilder.io/reference/markers/rbac) on Reconcile() & gen under /config/rbac by [controller-gen](https://book.kubebuilder.io/reference/controller-gen).

For each Kind, `*_[edit|view_role].yaml` role will be created.

```bash
$ make generate
```

**Manager** provides **shared resources** to all Controllers running within.

- A **client** for reading and writing resources
- A **cache** for reading resources from a local cache
- A **scheme** for registering all native and custom resources

`cmd/main.go`

### Run Project

```bash
$ make build IMG=myregistry/example:1.0.0
$ make deploy IMG=myregistry/example:1.0.0
```

