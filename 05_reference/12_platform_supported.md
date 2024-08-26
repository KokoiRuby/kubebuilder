Kubebuilder 默认提供多平台支持。

### How to define which platforms are supported

1. Build workload image to provide the support for other platform(s) 构建支持平台架构的工作负载镜像

```bash
# inspect image manifest 
$ docker manifest inspect myresgystry/example/myimage:vx.y.z
```

```json
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
    "manifests": [
        {
            "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
            "size": 739,
            "digest": "sha256:abcd1234...",
            "platform": {
                "architecture": "amd64",
                "os": "linux"
            }
        },
        {
            "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
            "size": 739,
            "digest": "sha256:efgh5678...",
            "platform": {
                "architecture": "arm64",
                "os": "linux"
            }
        }
    ]
}
```

2. Ensure that [node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) expressions are set to match the supported platforms 确保节点亲和性表达式匹配支持的平台架构。

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/arch
          operator: In
          values:
          - amd64
          - arm64
          - ppc64le
          - s390x
        - key: kubernetes.io/os
            operator: In
            values:
              - linux
```

```go
Template: corev1.PodTemplateSpec{
    ...
    Spec: corev1.PodSpec{
        Affinity: &corev1.Affinity{
            NodeAffinity: &corev1.NodeAffinity{
                RequiredDuringSchedulingIgnoredDuringExecution: &corev1.NodeSelector{
                    NodeSelectorTerms: []corev1.NodeSelectorTerm{
                        {
                            MatchExpressions: []corev1.NodeSelectorRequirement{
                                {
                                    Key:      "kubernetes.io/arch",
                                    Operator: "In",
                                    Values:   []string{"amd64"},
                                },
                                {
                                    Key:      "kubernetes.io/os",
                                    Operator: "In",
                                    Values:   []string{"linux"},
                                },
                            },
                        },
                    },
                },
            },
        },
        SecurityContext: &corev1.PodSecurityContext{
            ...
        },
        Containers: []corev1.Container{{
            ...
        }},
    },

```

### Producing projects that support multiple platforms

Use [`docker buildx`](https://docs.docker.com/build/buildx/) to cross-compile via emulation ([QEMU](https://www.qemu.org/)) to build the manager image. 构建多架构（multi-platform）镜像。

```bash
$ make docker-buildx IMG=myregistry/myoperator:v0.0.1
```

确保取消以下代码注释 `config/manager/manager.yaml`

```yaml
      # TODO(user): Uncomment the following code to configure the nodeAffinity expression
      # according to the platforms which are supported by your solution.
      # It is considered best practice to support multiple architectures. You can
      # build your manager image using the makefile target docker-buildx.
      # affinity:
      #   nodeAffinity:
      #     requiredDuringSchedulingIgnoredDuringExecution:
      #       nodeSelectorTerms:
      #         - matchExpressions:
      #           - key: kubernetes.io/arch
      #             operator: In
      #             values:
      #               - amd64
      #               - arm64
      #               - ppc64le
      #               - s390x
      #           - key: kubernetes.io/os
      #             operator: In
      #             values:
      #               - linux
```

### Workload image created by default

1. Manager `config/manager/manager.yaml`
   - binary `go build -a -o manager main.go`
   - image `make docker-build IMG=myregistry/myprojectname:<tag>`
2. [RBAC Proxy](https://github.com/brancz/kube-rbac-proxy) `config/default/manager_auth_proxy_patch.yaml` sidecar to protect the manager from attacks.