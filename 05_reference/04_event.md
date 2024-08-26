事件 Events 有助于让用户知晓对象目前处于什么状态以及之后如何响应该事件。

`kubectl describe <kind> <name>` & `kubectl get events`

根据 [Kubernetes APIs convention](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#events) Events 应该作为 Status 的一种补偿机制，而不是把所有信息都塞进去。

### Writing Events

```go
Event(object runtime.Object, eventtype, reason, message string)

// The following implementation will raise an event
r.Recorder.Event(cr, "Warning", "Deleting",
    fmt.Sprintf("Custom Resource %s is being deleted from the namespace %s",
        cr.Name,
        cr.Namespace))
```

- `object` 事件描述的对象。
- `eventtype` 事件类型 either *Normal* or *Warning*. ([More info](https://github.com/kubernetes/api/blob/6c11c9e4685cc62e4ddc8d4aaa824c46150c9148/core/v1/types.go#L6019-L6024))
- `reason` 事件产生的原因. It should be short and unique with `UpperCamelCase` format. The value could appear in *switch* statements by automation. ([More info](https://github.com/kubernetes/api/blob/6c11c9e4685cc62e4ddc8d4aaa824c46150c9148/core/v1/types.go#L6048))
- `message` 给事件消费的消息. ([More info](https://github.com/kubernetes/api/blob/6c11c9e4685cc62e4ddc8d4aaa824c46150c9148/core/v1/types.go#L6053))

`cmd/main.go` Manager 调用 GetRecorder 获取事件记录器。

```go
if err = (&controller.MyKindReconciler{
    Client:   mgr.GetClient(),
    Scheme:   mgr.GetScheme(),
    // Note that we added the following line:
    Recorder: mgr.GetEventRecorderFor("mykind-controller"),
}).SetupWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create controller", "controller", "MyKind")
    os.Exit(1)
}
```

Reconciler 结构体中内嵌组合。

```go
type MyKindReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    // embed
    Recorder record.EventRecorder
}
```

RBAC `make manifests` → `config/rbac/role.yaml`

```go
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
func (r *MyKindReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) { ... }
```



