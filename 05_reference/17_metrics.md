默认 controller-runtime 创建一个全局 Prometheus Registery 并给每个 controller 暴露一组[性能指标](https://book.kubebuilder.io/reference/metrics-reference)。

### Metrics Configuration

`config/default/kustomization.yaml`

```yaml
resources:
...
# [METRICS] Expose the controller manager metrics service.
- metrics_service.yaml

patches:
# [METRICS] The following patch will enable the metrics endpoint using HTTPS and the port :8443.
# More info: https://book.kubebuilder.io/reference/metrics
- path: manager_metrics_patch.yaml
  target:
    kind: Deployment
```

`cmd/main.go`

```go
metricsServerOptions := metricsserver.Options{
	BindAddress:   metricsAddr,
	SecureServing: secureMetrics,
	TLSOpts: tlsOpts,
}
```

### Metrics Protection

Kubebuilder applies AuthN & AuthZ to protect the metrics endpoints 即只有认证授权的用户才能访问。

Since v4.1.0 使用由 controller-runtime 提供的 [WithAuthenticationAndAuthorization](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/metrics/server)

`cmd/main.go` 使用 FilterProvider 执行 AuthN & AuthZ

```go
if secureMetrics {
  ...
  metricsServerOptions.FilterProvider = filters.WithAuthenticationAndAuthorization
}

```

`config/rbac/kustomization.yaml` 只有授权的用户 & Service Account 才能访问。

```yaml
resources:
# ...
- metrics_auth_role.yaml
- metrics_auth_role_binding.yaml
- metrics_reader_role.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: metrics-consumer
  namespace: system
spec:
  # Use the scaffolded service account name to allow authn/authz
  serviceAccountName: controller-manager
  containers:
  - name: metrics-consumer
    image: curlimages/curl:7.78.0
    command: ["/bin/sh"]
    args:
      - "-c"
      - >
        while true;
        do
          # Note here that we are passing the token obtained from the ServiceAccount to curl the metrics endpoint
          curl -s -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
          https://controller-manager-metrics-service.system.svc.cluster.local:8443/metrics;
          sleep 60;
        done
```

### Changes Recommended for Production

默认 metricsServerOptions 中 TLSOpts 使用自动生成的自签整数，只适合开发/测试，不推荐用于生产...

`config/prometheus/monitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
...
spec:
  endpoints:
    - path: /metrics
      ...
      tlsConfig:
        # Please use the following options for secure configurations:
        # caFile: /etc/metrics-certs/ca.crt
        # certFile: /etc/metrics-certs/tls.crt
        # keyFile: /etc/metrics-certs/tls.key
        insecureSkipVerify: true
```

1. Configure `CertDir`, `CertName`, and `KeyName` options to use your own certificates.
2. Configure Prometheus Monitoring Securely

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
...
spec:
  endpoints:
    - path: /metrics
      ...
      tlsConfig:
        # Please use the following options for secure configurations:
        caFile: /etc/metrics-certs/ca.crt
        certFile: /etc/metrics-certs/tls.crt
        keyFile: /etc/metrics-certs/tls.key
        # insecureSkipVerify: true
```

**By using Network Policy**

As basic firewall within K8s cluster at IP/Port level，但不处理 AuthN/AuthZ。

`config/default/kustomization.yaml` 取消注释

```yaml
resources:
- ../network-policy
```

**By exposing the metrics endpoint using HTTPS and CertManager**

`config/default/metrics_service.yaml`

`config/prometheus/monitor.yaml`

### Exporting Metrics for Prometheus

推荐 [kube-prometheus](https://github.com/coreos/kube-prometheus#installing) 

`config/default/kustomization.yaml` 取消注释，会创建 `ServiceMonitor` 负责导出 metrics。

```yaml
resources:
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
../prometheus
```

如果使用 Prometheus Operator 要注意 RBAC 只允许 default & kube-system 命名空间；如何[修改](https://github.com/prometheus-operator/kube-prometheus/blob/main/docs/monitoring-other-namespaces.md)？

或者使用 RBAC 赋予 Prometheus Operator 权限监控其他命名空间；[How](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md#enable-rbac-rules-for-prometheus-pods)？

### Publishing Additional Metrics

`controller-runtime/pkg/metrics` 声明全局变量 collector 然后在 init() 进行注册。

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    goobers = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "goobers_total",
            Help: "Number of goobers proccessed",
        },
    )
    gooberFailures = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "goober_failures_total",
            Help: "Number of failed goobers",
        },
    )
)

func init() {
    // Register custom metrics with the global prometheus registry
    metrics.Registry.MustRegister(goobers, gooberFailures)
}
```

