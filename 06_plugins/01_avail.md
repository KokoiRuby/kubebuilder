Since v3.0.0 提供了插件的初期支持。

### To scaffold the projects

#### `go/v4` 

只有显式声明才能使用该插件进行 scaffold = 初始化 proj

```bash
kubebuilder init 、
	--domain tutorial.kubebuilder.io \
	--repo tutorial.kubebuilder.io/project \
	--plugins=go/v4
```

[Migration](https://book.kubebuilder.io/migration/migration_guide_gov3_to_gov4) from `go/v3` → `go/v4`

### To add optional features

#### `grafana/v1-alpha`

to scaffold **Grafana Dashboards**

Prerequisites

- 必须使用 controller-runtime 暴露 [metrics](https://book.kubebuilder.io/reference/metrics-reference)。
- 这些 metrics 需要被 Prometheus 采集，[添加](https://grafana.com/docs/grafana/latest/datasources/add-a-data-source/) Grafana 的 datasource。
- 访问 Grafana

Artifacts: `grafana/controller-runtime-metrics.json`

```bash
# Initialize a new project with grafana plugin
kubebuilder init --plugins grafana.kubebuilder.io/v1-alpha

# Enable grafana plugin to an existing project
kubebuilder edit --plugins grafana.kubebuilder.io/v1-alpha
```

[Show case](https://book.kubebuilder.io/plugins/grafana-v1-alpha#show-case)

自定义 metrics `grafana/custom-metrics/config.yaml`

```yaml
---
customMetrics:
#  - metric: # Raw custom metric (required)
#    type:   # Metric type: counter/gauge/histogram (required)
#    expr:   # Prom_ql for the metric (optional)
#    unit:   # Unit of measurement, examples: s,none,bytes,percent,etc. (optional)
  - metric: memcached_operator_reconcile_total # Raw custom metric (required)
    type: counter                              # Metric type: counter/gauge/histogram (required)
    unit: none
  - metric: memcached_operator_reconcile_time_seconds_bucket
    type: histogram
```

重新生成 `grafana/custom-metrics/custom-metrics-dashboard.json` 导入 GUI。

```bash
$ kubebuilder edit --plugins grafana.kubebuilder.io/v1-alpha
```

#### `deploy-image/v1-alpha`

allows users to create [controllers](https://github.com/kubernetes-sigs/controller-runtime) and CR 用于部署和管理集群中的一个 Operand (image)。

```bash
kubebuilder create api \
	--group example.com \
	--version v1alpha1 \
	--kind Memcached \
	--image=memcached:1.6.15-alpine \
	--image-container-command="memcached,-m=64,modern,-v" \
	--image-container-port="11211" \
	--run-as-user="1001" \
	--plugins="deploy-image/v1-alpha"
```

ENV in `config/manager/manager.yaml`

```bash
$ export MEMCACHED_IMAGE="memcached:1.4.36-alpine"
$ make run
```

### To help projects to composite new solutions & plugins

#### `kustomize/v2`

Responsible for scaffolding all [kustomize](https://kustomize.io/) files under the `config/` directory.

```bash
$ kubebuilder init --plugins=kustomize/v2
```

#### `base/v4`

Responsible for scaffolding all files which specifically requires Golang.