Kubebuilder 使用 [controller-gen](https://sigs.k8s.io/controller-tools) 生成工具代码 & K8s obj YAML like CRD。

基于 [marker](https://book.kubebuilder.io/reference/markers) 注释 `// +...`。

Kubebuilder 提供 `make manifests` 生成 CRD under `config/crd/bases`。

### Validation

支持 [OpenAPI v3 schema](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.0.md#schemaObject) 声明式验证

marker 一般添加在字段 fields 或者类型 types

```go
type ToySpec struct {
    // +kubebuilder:validation:MaxLength=15      // 限制字符串的长度
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name,omitempty"`

    // +kubebuilder:validation:MaxItems=500      // 限制数组元素个数，是否允许重复
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:UniqueItems=true
    Knights []string `json:"knights,omitempty"`

    Alias   Alias   `json:"alias,omitempty"`
    Rank    Rank    `json:"rank"`
}

// +kubebuilder:validation:Enum=Lion;Wolf;Dragon  // 枚举
type Alias string

// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Maximum=3
// +kubebuilder:validation:ExclusiveMaximum=false // 数值允许最大值
type Rank int32


```

### Additional Printer Columns

`kubectl get` 显式额外的列信息 `// +kubebuilder:printcolumn`

```go
// +kubebuilder:printcolumn:name="Alias",type=string,JSONPath=`.spec.alias`
// +kubebuilder:printcolumn:name="Rank",type=integer,JSONPath=`.spec.rank`
type Toy struct {
    ...
}
```

### Subresources

`/status` 状态

```go
// +kubebuilder:subresource:status
type Toy struct {
    ...
    Status ToyStatus `json:"status,omitempty"`
}
```

`/scale` 资源扩缩容

```go
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas,selectorpath=.status.selector
type CustomSet struct {
    ...
    Spec   CustomSetSpec   `json:"spec,omitempty"`
    Status CustomSetStatus `json:"status,omitempty"`
}

type CustomSetSpec struct {
    Replicas *int32 `json:"replicas"`
}

type CustomSetStatus struct {
    Replicas int32 `json:"replicas"`
    Selector string `json:"selector"` // this must be the string form of the selector
}
```

### Multiple Version

标明哪个版本作为 storage 

```go
package v1

// +kubebuilder:storageversion

// CronJob is the Schema for the cronjobs API
type CronJob struct {
}
```

### Under the hood

kubebuilder 使用 controller-gen 生成 manifest，支持不同的输出规则 

```makefile
# Generate manifests for CRDs
# output:crd:artifacts rule to indicate CRD-related config
manifests: controller-gen
    $(CONTROLLER_GEN) rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```



