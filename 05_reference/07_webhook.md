Webhook 以阻塞的方式发送请求，通常某个事件触发后发送 HTTP 请求到其他应用的方式实现。

K8s 有 3 种类型

- [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)
- [authorization webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/) 
- [CRD conversion webhook](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion)

controller-runtime 支持 [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks) & [CRD conversion webhook](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion)，默认 v1。

```bash
$ kubebuilder create webhook ...
```

### Admission Webhooks

HTTP callback 接受 admission 请求 → 处理 → 返回 admission 响应。Type：

- **Mutating Admission Webhook** 持久前对 create/update/delete 请求进行修改 spec，比如注入默认值 or 注入 sidecar。
  - :anger: **不可以使用 mutating 修改对象状态 status or 设置默认值**
- **Validating Admission Webhook** 持久前对 create/update/delete 请求进行验证 spec；注：**在所有 mutating 执行之后才执行**。

对于 core types，kubebuilder 不提供 scaffolding，需要使用 controller-runtme 处理，要实现 `admission.Handler` 接口。

```go
type podAnnotator struct {
    Client  client.Client
    decoder *admission.Decoder
}

func (a *podAnnotator) Handle(ctx context.Context, req admission.Request) admission.Response {
    pod := &corev1.Pod{}
    err := a.decoder.Decode(req, pod)
    if err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }

    // mutate the fields in pod

    marshaledPod, err := json.Marshal(pod)
    if err != nil {
        return admission.Errored(http.StatusInternalServerError, err)
    }
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}

```

marker

```go
// +kubebuilder:webhook:
path=/mutate-v1-pod,
mutating=true,
failurePolicy=fail,
groups="",
resources=pods,
verbs=create;update,
versions=v1,
name=mpod.kb.io
```

`cmd/main.go` 注册 handler into webhook server。**注：路径需要匹配 marker 中的 path**。

```go
mgr.GetWebhookServer().Register(
    "/mutate-v1-pod", 
    &webhook.Admission{
        Handler: &podAnnotator{Client: mgr.GetClient()}
        // if need client & decoder
        Client:   mgr.GetClient(),
        decoder:  admission.NewDecoder(mgr.GetScheme()),
    },
)
```

