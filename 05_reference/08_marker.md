Kubebuilder 使用 [controller-gen](https://book.kubebuilder.io/reference/controller-gen) 生成 utitlity 代码 & K8s YAML，由 **marker comments** 控制代码生成。

vs.

`// +optional` 通用的标记，用来表示某个字段是可选的。在生成 CRD 的过程中，这个标记会将该字段设定为可选字段。也就是说，当用户创建或更新 CRD 对象时，该字段可以省略。

`// +kubebuilder:validation:Optional` 用于更明确地在 OpenAPI 模式中指定字段的可选性，在 CRD 验证的细节上提供了更精确的控制。

### Make

`make manifests` generates Kubernetes object YAML, like [CustomResourceDefinitions](https://book.kubebuilder.io/reference/markers/crd), [WebhookConfigurations](https://book.kubebuilder.io/reference/markers/webhook), and [RBAC roles](https://book.kubebuilder.io/reference/markers/rbac).

`make generate` generates code, like [runtime.Object/DeepCopy implementations](https://book.kubebuilder.io/reference/markers/object).

### Syntax

**Empty** boolean flag as enabler `// +kubebuilder:validation:Optional`

**Anonymous** take a single value as arg. `// +kubebuilder:validation:MaxItems=2`

**Multi-option** take one or more named args. `// +kubebuilder:printcolumn:JSONPath=".status.replicas",name=Replicas,type=string`

Quote may be omitted.

`{,}` or `;` for multiple strings.

map is surrounded by `{}`.

### [CRD Generation](https://book.kubebuilder.io/reference/markers/crd)

### [CRD Validation](https://book.kubebuilder.io/reference/markers/crd-validation#crd-validation)

### [CRD Processing](https://book.kubebuilder.io/reference/markers/crd-processing#crd-processing)

### [Webhook](https://book.kubebuilder.io/reference/markers/webhook#webhook)

### [Object/DeepCopy](https://book.kubebuilder.io/reference/markers/object#objectdeepcopy)

### [RBAC](https://book.kubebuilder.io/reference/markers/rbac#rbac)