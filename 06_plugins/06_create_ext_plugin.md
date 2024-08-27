一个**外部插件**是一个可执行（可以是任何语言）实现了 Kubebuilder 的执行模式，Kubebuilder CLI 加载指定路径下的外部插件，并通过 stdin/out 进行交互。

- 当你想要在 Kubebuilder 默认 scaffolding 添加 helpers/addons
- 当你想要设计自定义的 layouts
- 当你想要寻找其他语言实现的插件

[PluginRequest](https://github.com/kubernetes-sigs/kubebuilder/blob/master/pkg/plugin/external/types.go#L23)

- 封装所有通过 CLI 收集的数据 + plugin 链上所有此前执行的插件
- Kubebuilder 将序列化的 PluginRequest (JSON) 通过 stdin 传递给外部插件

```json
{
    "apiVersion": "v1alpha1",
    "args": ["--domain", "my.domain"],
    "command": "init",
    "universe": {}
}
```

[PluginResponse](https://github.com/kubernetes-sigs/kubebuilder/blob/master/pkg/plugin/external/types.go#L42)

- 表示插件作用后，项目更新后的状态；同样以序列化 JSON 的形式通过 stdout 返回。
- :anger: **不应该直接 echo/print 任何内容到 stdout！**

```json
{
    "apiVersion": "v1alpha1",
    "command": "init",
    "metadata": {
        "description": "The `init` subcommand is meant to initialize a project via Kubebuilder. It scaffolds a single file: `initFile`",
        "examples": "kubebuilder init --plugins sampleexternalplugin/v1 --domain my.domain"
    },
    "universe": {
        "initFile": "A simple file created with the `init` subcommand"
    },
    "error": false,
    "errorMsgs": []
}
```

### How

**Prereqsuisites**

- kubebuilder CLI > 3.11.0
- An executable for the external plugin
- Configuration of the external plugin’s path `${EXTERNAL_PLUGINS_PATH}` ENV

```bash
$HOME/.config/kubebuilder/plugins/${name}/${version}/${name}
```

**Subcommands**

- `init`: project initialization.

- `create api`: scaffold Kubernetes API definitions.

- `create webhook`: scaffold Kubernetes webhooks.

- `edit`: update the project configuration.

  (Optional)

- `metadata`: add customized plugin description and examples when a `--help` flag is specified.

- `flags`: specify valid flags for Kubebuilder to pass to the external plugin.

**Configurations**

```bash
$ export EXTERNAL_PLUGINS_PATH = <custom-path>
$ kubebuilder [subcommand]
```

### TODO

A [sample external plugin written in Go](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/simple-external-plugin-tutorial/testdata/sampleexternalplugin/v1)