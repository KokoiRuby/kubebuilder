### CLI

Plugins are run using a [`CLI`](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/pkg/cli) object, which maps a plugin type to a subcommand and calls that plugin’s methods. 

映射一个插件类型 → subcommand 然后调用插件的方法。

CLI 负责管理 `PROJECT` 文件（保存项目配置用于 scaffold）

```go
package cli

import (
    log "github.com/sirupsen/logrus"
    "github.com/spf13/cobra"

    "sigs.k8s.io/kubebuilder/v4/pkg/cli"
    cfgv3 "sigs.k8s.io/kubebuilder/v4/pkg/config/v3"
    "sigs.k8s.io/kubebuilder/v4/pkg/plugin"
    kustomizecommonv2 "sigs.k8s.io/kubebuilder/v4/pkg/plugins/common/kustomize/v2"
    "sigs.k8s.io/kubebuilder/v4/pkg/plugins/golang"
    deployimage "sigs.k8s.io/kubebuilder/v4/pkg/plugins/golang/deploy-image/v1alpha1"
    golangv4 "sigs.k8s.io/kubebuilder/v4/pkg/plugins/golang/v4"

)

var (
    // The following is an example of the commands
    // that you might have in your own binary
    commands = []*cobra.Command{
        myExampleCommand.NewCmd(),
    }
    alphaCommands = []*cobra.Command{
        myExampleAlphaCommand.NewCmd(),
    }
)

// GetPluginsCLI returns the plugins based CLI configured to be used in your CLI binary
func GetPluginsCLI() (*cli.CLI) {
    // Bundle plugin which built the golang projects scaffold by Kubebuilder go/v4
    gov3Bundle, _ := plugin.NewBundleWithOptions(plugin.WithName(golang.DefaultNameQualifier),
        plugin.WithVersion(plugin.Version{Number: 3}),
        plugin.WithPlugins(kustomizecommonv2.Plugin{}, golangv4.Plugin{}),
    )


    c, err := cli.New(
        // Add the name of your CLI binary
        cli.WithCommandName("example-cli"),

        // Add the version of your CLI binary
        cli.WithVersion(versionString()),

        // Register the plugins options which can be used to do the scaffolds via your CLI tool. 
        // See that we are using as example here the plugins which are implemented and provided by Kubebuilder
        cli.WithPlugins(
            gov3Bundle,
            &deployimage.Plugin{},
        ),

        // Defines what will be the default plugin used by your binary. 
        // means that will be the plugin used if no info be provided such as when the user runs `kubebuilder init`
        cli.WithDefaultPlugins(cfgv3.Version, gov3Bundle),

        // Define the default project configuration version 
        // which will be used by the CLI when none is informed by --project-version flag.
        cli.WithDefaultProjectVersion(cfgv3.Version),

        // Adds your own commands to the CLI
        cli.WithExtraCommands(commands...),

        // Add your own alpha commands to the CLI
        cli.WithExtraAlphaCommands(alphaCommands...),

        // Adds the completion option for your CLI
        cli.WithCompletion(),
    )
    if err != nil {
        log.Fatal(err)
    }

    return c
}

// versionString returns the CLI version
func versionString() string {
    // return your binary project version
}
```

```bash
# Initialize a project with the default Init plugin, "go.example.com/v1".
# This key is automatically written to a PROJECT config file.
$ my-bin-builder init
$ my-bin-builder init --plugins ansible

# Create an API and webhook with "go.example.com/v1" CreateAPI and
# CreateWebhook plugin methods. This key was read from the config file.
$ my-bin-builder create api [flags]
$ my-bin-builder create webhook [flags]
```

### Plugins

实现 [Plugin Interface](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/pkg/plugin#Plugin) 来创建一个新的 Plugin。

顶级 Plugin `Base` 实现了 `SubcommandMetadata` 接口，可以通过 CLI 运行，可选设置 help text。

每一个 Plugin 都配置了其中一个 CLI 的指令

- init
- create api
- create webhook

Plugin 通过 `<name>/<version>` 进行标识。

**使用 Plugin**

1. `kubebuilder init --plugins=<plugin key>`
2. `layout: <plugin key>` (default is: go.kubebuilder.io/v[X]) in `PROJECT`

**Plugin 命名**遵循 DNS1123

**版本**：Version() 方法返回 [`plugin.Version`](https://pkg.go.dev/sigs.k8s.io/kubebuilder/v4/pkg/plugin#Version) 对象包含了一个 integer value & stage string (alpha or beta)

**Bundle**

```go
   // see that will be like myplugin.example/v1`
  myPluginBundle, _ := plugin.NewBundle(plugin.WithName(`<plugin-name>`),
  		plugin.WithVersion(`<plugin-version>`),
        plugin.WithPlugins(pluginA.Plugin{}, pluginB.Plugin{}, pluginC.Plugin{}),
    )

```

```bash
# subcommand of A → B → C
$ kubebuider init --plugins=myplugin.example/v1

```

