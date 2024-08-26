[`controller-runtime/pkg/envtest`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest) 通过配置并启动 API Server & etcd 实例（无 kubelet/controller-manager 及其他组件）对 controller 测试。

### Installation

下载 binaries → `./bin`

```bash
$ make envtest
```

Makefile

```makefile
ENVTEST_K8S_VERSION = 1.31.0
ENVTEST_VERSION ?= release-0.19
...
.PHONY: envtest
envtest: $(ENVTEST) ## Download setup-envtest locally if necessary.
$(ENVTEST): $(LOCALBIN)
    $(call go-install-tool,$(ENVTEST),sigs.k8s.io/controller-runtime/tools/setup-envtest,$(ENVTEST_VERSION))
```

### Writing tests

[ginkgo](https://github.com/onsi/ginkgo) test suite

```go
import sigs.k8s.io/controller-runtime/pkg/envtest

// specify testEnv configuration 声明配置
testEnv = &envtest.Environment{
    CRDDirectoryPaths: []string{filepath.Join("..", "config", "crd", "bases")},
}

// start testEnv 启动测试
cfg, err = testEnv.Start()

// write test logic 测试逻辑

// stop testEnv 停止测试
err = testEnv.Stop()
```

### Configuring your test control plane

```bash
./bin/k8s/
└── x.y.z-darwin-amd64
    ├── etcd
    ├── kube-apiserver
    └── kubectl
```

`suite_test.go` 代码中配置环境变量

```go
var _ = BeforeSuite(func(done Done) {
    Expect(os.Setenv("TEST_ASSET_KUBE_APISERVER", "../bin/k8s/x.y.z-darwin-amd64/kube-apiserver")).To(Succeed())
    Expect(os.Setenv("TEST_ASSET_ETCD", "../bin/k8s/x.y.z-darwin-amd64/etcd")).To(Succeed())
    Expect(os.Setenv("TEST_ASSET_KUBECTL", "../bin/k8s/x.y.z-darwin-amd64/kubectl")).To(Succeed())
    // OR
    Expect(os.Setenv("KUBEBUILDER_ASSETS", "../bin/k8s/x.y.z-darwin-amd64")).To(Succeed())

    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))
    testenv = &envtest.Environment{}

    _, err := testenv.Start()
    Expect(err).NotTo(HaveOccurred())

    close(done)
}, 60)

var _ = AfterSuite(func() {
    Expect(testenv.Stop()).To(Succeed())

    Expect(os.Unsetenv("TEST_ASSET_KUBE_APISERVER")).To(Succeed())
    Expect(os.Unsetenv("TEST_ASSET_ETCD")).To(Succeed())
    Expect(os.Unsetenv("TEST_ASSET_KUBECTL")).To(Succeed())

})
```

**Modifiy flags of API Server**

```go
customApiServerFlags := []string{
    "--secure-port=6884",
    "--admission-control=MutatingAdmissionWebhook",
}

apiServerFlags := append([]string(nil), envtest.DefaultKubeAPIServerFlags...)
apiServerFlags = append(apiServerFlags, customApiServerFlags...)

testEnv = &envtest.Environment{
    CRDDirectoryPaths: []string{filepath.Join("..", "config", "crd", "bases")},
    KubeAPIServerFlags: apiServerFlags,
}

```

### Testing considerations

注意：测试并**未提供内建 controllers**，所以测试的结果可能会和真实集群的表现存在一定的差异...没有 GC，即便设置了 OwnerReference，由于没有内建 controller 所以对象不会被删除。

∴ 在测试删除时，应该测试其 ownership 而不是对象是否存在。

```go
expectedOwnerReference := v1.OwnerReference{
    Kind:       "MyCoolCustomResource",
    APIVersion: "my.api.example.com/v1beta1",
    UID:        "d9607e19-f88f-11e6-a518-42010a800195",
    Name:       "userSpecifiedResourceName",
}
Expect(deployment.ObjectMeta.OwnerReferences).To(ContainElement(expectedOwnerReference))
```

### Namespace usage limitation

EnvTest 不支持删除命名空间；删除行为会被视为成功，置为 Terminating 状态且不会被回收，重新创建会失败...进而导致 reconciler 会持续不断 reconcile 残余的对象，除非他们被删除了...

解决：每个测试创建一个新的命名空间。

如果删除命名空间很困难，可以使 reconciler 忽略来自已测试命名空间的 reconcile 请求。

```go
type MyCoolReconciler struct {
    client.Client
    ...
    Namespace     string  // restrict namespaces to reconcile
}
func (r *MyCoolReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = r.Log.WithValues("myreconciler", req.NamespacedName)
    // Ignore requests for other namespaces, if specified
    if r.Namespace != "" && req.Namespace != r.Namespace {
        return ctrl.Result{}, nil
    }
```

### [Cert-Manager and Prometheus options](https://book.kubebuilder.io/reference/envtest#cert-manager-and-prometheus-options)

