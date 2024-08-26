KubeBuilder project 配置，所有的项目都通过 CLI 生成，以及 `PROJECT` 文件在项目的根目录下。

`PROJECT` （带版本描述）文件描述了所有的**插件 & 输入数据**用于生成项目 & API。

### Why need to store?

- 检查插件是否能在现有的插件链上生效/兼容
- 基于当前的配置能执行什么/不能执行什么操作
- 验证数据是否在操作中被使用

### [Layout Definition](https://book.kubebuilder.io/reference/project-config#layout-definition)