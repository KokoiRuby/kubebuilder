To **test** your controllers, you will need to use the tarballs containing the required binaries. [Release](https://github.com/kubernetes-sigs/controller-tools/blob/main/envtest-releases.yaml) to download.

```
./bin/k8s/
└── x.y.z-darwin-amd64
    ├── etcd
    ├── kube-apiserver
    └── kubectl
```

Auto downloaded & properly configured for ur proj.

```bash
$ make envtest
# or
$ make test
```

