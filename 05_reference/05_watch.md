> 要管理对象就必须要通过监视知晓对象的状态

每当一个受监视对象的动作 create/update/delete 被触发时，`Reconcile()` 就会被调用。

### Watching Operator Managed Resources

These resources are created and managed **by the same operator** as the resource watching them.

通过 Manager 的 `Owns()` 可以简单实现监视。

```go
// SetupWithManager sets up the controller with the Manager.
func (r *SimpleDeploymentReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1.SimpleDeployment{}).
        Owns(&kapps.Deployment{}).
        Complete(r)
}
```

设置 controller reference in `Reconcile()`。

```go
if err := controllerutil.SetControllerReference(simpleDeployment, deployment, r.scheme); err != nil {
    return ctrl.Result{}, err
}
```

### Watching Externally Managed Resources

These resources could be manually created, or managed **by other operators/controllers or the Kubernetes control plane**.

**引用外部资源**

- Ingress → service
- Pods → ConfigMaps, Secrets & Volumes
- Deployments/Services → Pods

Example: The `ConfigDeployment`’s purpose is to manage a `Deployment` whose pods are always using the latest version of a `ConfigMap`.

`api.go`

```go
type ConfigDeploymentSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // Name of an existing ConfigMap in the same namespace, to add to the deployment
    // +optional
    ConfigMap string `json:"configMap,omitempty"`
}
```

`controller.go`

```go
import (
	// Required for Watching
    "k8s.io/apimachinery/pkg/fields"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/builder"
    "sigs.k8s.io/controller-runtime/pkg/handler"   
    "sigs.k8s.io/controller-runtime/pkg/predicate" 
    "sigs.k8s.io/controller-runtime/pkg/reconcile" 
    "sigs.k8s.io/controller-runtime/pkg/source" 
)
```

RBAC 需要管理 Deployments + status & get/list/watch on ConfigMaps。

```go
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get
// +kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch
```

我们需要添加一个注释到 PodTemplate 以追踪引用 ConfigMap 的版本情况；若版本发生变化，那么会触发一次滚动更新。

```go
// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
func (r *ConfigDeploymentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    ...
    // your logic here
    var configMapVersion string
    if configDeployment.Spec.ConfigMap != "" {
        configMapName := configDeployment.Spec.ConfigMap
        foundConfigMap := &corev1.ConfigMap{}
        err := r.Get(ctx, types.NamespacedName{Name: configMapName, Namespace: configDeployment.Namespace}, foundConfigMap)
        if err != nil {
            // If a configMap name is provided, then it must exist
            // You will likely want to create an Event for the user to understand why their reconcile is failing.
            return ctrl.Result{}, err
        }

        // Hash the data in some way, or just use the version of the Object
        configMapVersion = foundConfigMap.ResourceVersion
    }

    // Logic here to add the configMapVersion as an annotation on your Deployment Pods.

    return ctrl.Result{}, nil
}
```

注：监视的 ConfigMaps 并不被 ConfigDeployments 拥有，需要提供一种自定义的监视。

我们先要获取到 ConfigDeployments 引用的 ConfigMap 名。

```go
if err := mgr.GetFieldIndexer().IndexField(context.Background(), &appsv1.ConfigDeployment{}, configMapField, func(rawObj client.Object) []string {
    // Extract the ConfigMap name from the ConfigDeployment Spec, if one is provided
    configDeployment := rawObj.(*appsv1.ConfigDeployment)
    if configDeployment.Spec.ConfigMap == "" {
        return nil
    }
    return []string{configDeployment.Spec.ConfigMap}
}); err != nil {
    return err
}
```

使用 `Watches()` 方法

- 接受一个 Kind
- 一个映射函数，将 ConfigMaps 对象转换成一组 ConfigDeployments 的 Reconcile 亲求
- 一组 options 用于监视 ConfigMaps

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&appsv1.ConfigDeployment{}).
    Owns(&kapps.Deployment{}).
    Watches(
        &corev1.ConfigMap{},
        handler.EnqueueRequestsFromMapFunc(r.findObjectsForConfigMap),
        builder.WithPredicates(predicate.ResourceVersionChangedPredicate{}),
    ).
    Complete(r)
```

```go
// ConfigMap → []reconcile.Request
func (r *ConfigDeploymentReconciler) findObjectsForConfigMap(ctx context.Context, configMap client.Object) []reconcile.Request {
    attachedConfigDeployments := &appsv1.ConfigDeploymentList{}
    listOps := &client.ListOptions{
        FieldSelector: fields.OneTermEqualSelector(configMapField, configMap.GetName()),
        Namespace:     configMap.GetNamespace(),
    }
    err := r.List(ctx, attachedConfigDeployments, listOps)
    if err != nil {
        return []reconcile.Request{}
    }
	// Deployment with configmap found
    requests := make([]reconcile.Request, len(attachedConfigDeployments.Items))
    for i, item := range attachedConfigDeployments.Items {
        // build request
        requests[i] = reconcile.Request{
            NamespacedName: types.NamespacedName{
                Name:      item.GetName(),
                Namespace: item.GetNamespace(),
            },
        }
    }
    return requests
}
```

