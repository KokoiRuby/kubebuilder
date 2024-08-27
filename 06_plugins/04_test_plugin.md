Plugin 测试

1. E2E
2. 生成 sample 项目 under `./testdata/`

### E2E

`TestContext` in [Kubebuilder/v3/test/e2e/utils](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/test/e2e/utils)

[NewTestContext](https://github.com/kubernetes-sigs/kubebuilder/blob/v3.7.0/test/e2e/utils/test_context.go#L51) 提供

- 测试项目用临时目录
- 临时 controller-manager 镜像
- [kubectl 执行方法](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/test/e2e/utils#Kubectl)
- 可执行 cli 

For more info [operator-sdk e2e tests](https://github.com/operator-framework/operator-sdk/tree/master/test/e2e/go), [kubebuiler e2e tests](https://github.com/kubernetes-sigs/kubebuilder/tree/master/test/e2e/v3)

### Generate Test Samples

Kubebuilder 默认生成 sample projects。

简单通过 TextContext 测试 plugin 创建 sample。

```go
// kbc is a instance of TestContext

// init proj
By("initializing a project")
err = kbc.Init(
    "--plugins", "go/v4",
    "--project-version", "3",
    "--domain", kbc.Domain,
    "--fetch-deps=false",
)
ExpectWithOffset(1, err).NotTo(HaveOccurred())

// define api
By("creating API definition")
err = kbc.CreateAPI(
    "--group", kbc.Group,
    "--version", kbc.Version,
    "--kind", kbc.Kind,
    "--namespaced",
    "--resource",
    "--controller",
    "--make=false",
)
ExpectWithOffset(1, err).NotTo(HaveOccurred())

// scaffold webhook
By("scaffolding mutating and validating webhooks")
err = kbc.CreateWebhook(
    "--group", kbc.Group,
    "--version", kbc.Version,
    "--kind", kbc.Kind,
    "--defaulting",
    "--programmatic-validation",
)
ExpectWithOffset(1, err).NotTo(HaveOccurred())
```



