Kubebuilder 扩展

1. 重用现有插件，以这种方式将 Kubebuilder 视为 lib。
2. 创建一个外部插件，独立单一的二进制文件。

UC

- 当你想要创建一个插件支持其他语言
- 当你想要创建 helper 以及在 Kubebuilder 上做集成
- 当你想要自定义项目 layouts

通过 `Bundle Plugin` 创建其他语言的插件，然后为 sub-commands `create api` & `create webhook` 创建对应语言的实现。

```go
 mylanguagev1Bundle, _ := plugin.NewBundle(plugin.WithName(language.DefaultNameQualifier),
     plugin.WithVersion(plugin.Version{Number: 1}),
     plugin.WithPlugins(kustomizecommonv1.Plugin{}, mylanguagev1.Plugin{}), // extend the common base from Kubebuilder
     // your plugin language which will do the scaffolds for the specific language on top of the common base
)
```

[machinery pkg](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/pkg/machinery#section-documentation)

- 定义文件 IO 行为
- 添加 marker 到 scaffolded 文件中
- 定义 scaffolding 模板

**Combo of multiple plugins**

对于频繁使用的插件，命令行指定会很笨重；可以定义一个方法按序调用这些插件方法。[Example](https://github.com/kubernetes-sigs/kubebuilder/blob/master/pkg/plugins/golang/deploy-image/v1alpha1/scaffolds/api.go#L77-L98)。



