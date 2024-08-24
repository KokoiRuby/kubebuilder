### [Installation](https://book.kubebuilder.io/quick-start#installation)

### [Create a Project](https://book.kubebuilder.io/quick-start#create-a-project)

```bash
# $GOPATH/src/github.com/kubebuilder/00_quick_start/
$ mkdir -p ./projects/guestbook
$ cd ~/projects/guestbook
$ kubebuilder init \
	--domain my.domain \
	--repo my.domain/guestbook
```

### [Create an API](https://book.kubebuilder.io/quick-start#create-an-api)

Press `y` if u want to create:

- Resource `api/v1/guestbook_types.go`
- Controller `internal/controllers/guestbook_controller.go`

```bash
$ kubebuilder create api \
	--group webapp \
	--version v1 \
	--kind Guestbook
```

Generate CR & CRD to `config/crd/bases` to `config/samples`

```bash
$ make manifests
```

### [Test It Out](https://book.kubebuilder.io/quick-start#test-it-out)

Install CRD into the ([KIND](https://sigs.k8s.io/kind)) cluster.

```bash
$ make install
```

Run controller in the foreground.

```bash
$ make run
```

### [Install Instances of Custom Resources](https://book.kubebuilder.io/quick-start#install-instances-of-custom-resources)

```bash
# -k specifiy dir
$ kubectl apply -k config/samples/
```

### [Run It On the Cluster](https://book.kubebuilder.io/quick-start#run-it-on-the-cluster)

```bash
# build & deploy controller
$ make docker-build docker-push IMG=<some-registry>/<project-name>:tag
$ make deploy IMG=<some-registry>/<project-name>:tag
```

### Uninstall

```bash
# uninstall CRD & uninstall controller
$ make uninstall
$ make undeploy
```

