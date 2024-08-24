The job of the *CronJob* controller is to **run one-off tasks** on the K8s cluster at regular intervals. It does this by building on top of the *Job* controller, whose task is to run one-off tasks once, seeing them to completion.

```bash
$ kubebuilder init \
	--domain tutorial.kubebuilder.io \
	--repo tutorial.kubebuilder.io/project
```

### 1. Basic Project

`go.mod` go module file to manage dep.

`Makefile` makes targets for building & deployment.

`PROJECT`: project meta.

`config/default`: [kustomize](https://github.com/kubernetes-sigs/kustomize) base (customize raw, template-free YAML files).

`config/manager` launch controller as pods in the cluster.

`config/rbac`: permission control.

### 2. Main

`cmd/main.go` entrypoint.

Import: core [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime?tab=doc) & [Zap](https://pkg.go.dev/go.uber.org/zap) for logging.

每一个 controller 都需要一个 **Scheme** 负责 GVK & Type 的映射，即反序列化。

```go
var (
	scheme   = runtime.NewScheme()
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    // +kubebuilder:scaffold:scheme
}
```

Setup metrics flags

Instantiate a manager to keep track of all controllers, setting up shared caches & clients.

Run the manager. (Not yet) 

```go
metricsServerOptions := metricsserver.Options{
	BindAddress:   metricsAddr,
	SecureServing: secureMetrics,
	// TODO(user)
	TLSOpts: tlsOpts,
}

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    ...
}
                            
// +kubebuilder:scaffold:builder
// TODO(user)  
```

`cache.Options{}` 可限制 controller 监视某些特定 namespace 中的资源。

同时还要注意限制 RABC - [Cluster]Role[Binding]

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    ...
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{
            namespace: {},
        },
    },
    ...
}                           
```

```go
var namespaces []string
defaultNamespaces := make(map[string]cache.Config)

// cache conf per ns
for _, ns := range namespaces {
    defaultNamespaces[ns] = cache.Config{}
}

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
	...
    Cache: cache.Options{
        DefaultNamespaces: defaultNamespaces,
    },
    ...
}
```

### 3. GVK

K8s API Group 表示相关功能的结合。每一个 Group 都有若干 Version。

每一个 GV 都包含若干 API Types， aka Kinds；Kind 因版本不同会有变化。

Resource 是 Kind 的视图（从使用者的视角）存在 1-to-1 映射关系。e.g. `pods` Resource → `Pod` Kind。

**GVK = Refer to a Kind in a particular GV in a form of struct type in Go.**

Go 结构体可用于生成 CRD 包含 schema 和跟踪变化。

GVK → type 映射关系由 Scheme 负责维护和转换。

### 4. Adding a new API

`--domain` **tutorial.kubebuilder.io**

最终 GV → batch.**tutorial.kubebuilder.io**/v1

```bash
$ kubebuilder create api \
	--group batch \
	--version v1 \
	--kind CronJob
```

`/api/v1/cronjob_types.go`

Marker `// +kubebuilder:object:root=true` 告诉 `object` generator 该结构体类型用于表示一个 Kind，并为我们实现 `runtime.Object` 接口。

```go
// +kubebuilder:object:root=true
type CronJob struct {
	// per kind
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	
    // desired
	Spec   CronJobSpec   `json:"spec,omitempty"`
    // actual
	Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type CronJobList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CronJob `json:"items"`
}
```

将 types 添加至 API Group，即注册到 Scheme。

```go
func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

### 5. Design an API

Rules

- fields in camelCase
- JSON struct tag
- omitempty 表示若值为空，那么就不执行序列化
- 只接收 3 中数值类型
  - `int[32|64]` for integers
  - `resource.Quantity` for decimal
    - 2m  = 0.002
    - 2Ki = 2048
    - 2K  = 2000
    - 2.5 = 2500m
- 时间 `metav1.Time` 对 `time.tim`e 的封装，只支持固定序列化格式。

**CronJobSpec**

- A schedule
- A template for the Job to run
- Extra
  - A deadline for starting jobs
  - What to do if multiple jobs would run at once
  - A way to pause the running of a CronJob in case something is wrong
  - Limits on old job history

```go
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
	AllowConcurrent ConcurrencyPolicy = "Allow"
	ForbidConcurrent ConcurrencyPolicy = "Forbid"
	ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```

```go
type CronJobSpec struct {
	// +kubebuilder:validation:MinLength=0
	// https://en.wikipedia.org/wiki/Cron.
	Schedule string `json:"schedule"`

	// +kubebuilder:validation:Minimum=0
	// +optional
	StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

	ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

	// +optional
	Suspend *bool `json:"suspend,omitempty"`

	JobTemplate batchv1.JobTemplateSpec `json:"jobTemplate"`

	// +kubebuilder:validation:Minimum=0
	// +optional
	SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

	// +kubebuilder:validation:Minimum=0
	// +optional
	FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

**CronJobStatus**

```go
type CronJobStatus struct {
	// A list of pointers to currently running jobs.
	// +optional
	Active []corev1.ObjectReference `json:"active,omitempty"`

	// +optional
	LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```

`// +kubebuilder:subresource:status` 标识为该资源生成 status 子资源。

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
    ...
}
```

**Misc**

`api/v1/`

- `groupversion_info.go` common meta about GV.

```go
// package-level markers

// +kubebuilder:object:generate=true
// +groupName=batch.tutorial.kubebuilder.io
package v1
```

```go
// var for scheme
var (
	GroupVersion = schema.GroupVersion{Group: "batch.tutorial.kubebuilder.io", Version: "v1"}
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}
	AddToScheme = SchemeBuilder.AddToScheme
)
```

- `zz_generated.deepcopy.go` autogenerated implementation of `runtime.Object`.

```go
func (in *CronJob) DeepCopyObject() runtime.Object
```

### 6. What's in a controller?

controller 的作用：**对于任何单一对象，驱动 actual → desired 状态，即 Reconciliation**

每一个 controller 关注一个 root Kind，该 root Kind 由其他 Kinds 实现。

**Reconciler** 用于实现 Reconciliation，它接受一个对象，返回是否需要重试。

`/internal/controller/*_controller.go`

```go
type CronJobReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}
```

RBAC [markers](https://book.kubebuilder.io/reference/markers/rbac)

```go
// // +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=...,verbs=...
```

Generate under `/config/rbac/`

```bash
$ make manifests
```

**Business Logic**

`ctrl.Request` 包含了需要 reconcile 对象的所有必要信息，通常是 obj-key 即 namespace/name。

```go
func (r *CronJobReconciler) Reconcile(
    ctx context.Context, 
    req ctrl.Request)
(ctrl.Result, error) {
    // logging
    _ = log.FromContext(ctx)

    // your logic here

    // indicate we’ve successfully reconciled this object, no need to try again
    return ctrl.Result{}, nil
}
```

添加 Reconciler 到 Manager，随 Manager 启动运行。

```go
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.CronJob{}).
		Complete(r)
}
```

### 7. [Implementing a controller](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation#implementing-a-controller)

1. Load the named CronJob
2. List all active jobs, and update the status
3. Clean up old jobs according to the history limits
4. Check if we’re suspended (and don’t do anything else if we are)
5. Get the next scheduled run
6. Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy
7. Requeue when we either see a running job (done automatically) or it’s time for the next scheduled run.

`cmd/main.go`

```go
// add CRD into scheme
// batchv1/job already insde
var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)
```

### 8. Implementing defaulting/validating webhooks

Impl `Defaulter` and/or `Validator` interface.

Kubebuilder takes care of the rest for you, such as

1. 创建 webhook server
2. 确保 server 添加进 manager
3. 创建 webhook handler
4. 注册每个 handler 到 path

`--defaulting` 告诉 Kubebuilder 为所创建的 webhook 实现默认值设定功能。

`--programmatic-validation` 告诉 Kubebuilder 为所创建的 webhook 实现验证功能。

```bash
$ kubebuilder create webhook \
	--group batch \
	--version v1 \
	--kind CronJob \
	--defaulting \
	--programmatic-validation
```

`project/api/v1/cronjob_webhook.go`

```go
// logger
var cronjoblog = logf.Log.WithName("cronjob-resource")

// set up with manager
func (r *CronJob) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}
```

[marker](https://book.kubebuilder.io/reference/markers/webhook) for defaulting

```go
// +kubebuilder:webhook:
path=/mutate-batch-tutorial-kubebuilder-io-v1-cronjob, // 声明 API Server 连接到该 webhook 的路径
mutating=true,                                         // 标识这是 mutating webhook
failurePolicy=fail,                                    // 若 API Server 无法连接到 webhook 则拒绝该对象
sideEffects=None,                                      // 是否要调用带副作用的 webhook，只影响 dryrun & kubectl diff
groups=batch.tutorial.kubebuilder.io,                  // webhook 接受请求所属的 API group
resources=cronjobs,                                    // webhook 接受请求所属的资源
verbs=create;update,                                   // 支持的操作
versions=v1,                                           // webhook 接受请求所属的资源版本
name=mcronjob.kb.io,                                   // webhook 配置名
admissionReviewVersions=v1                             // admission review 版本
```

Use `webhook.Defaulter` interface to set defaults to CRD.

```go
func (r *CronJob) Default() {
	cronjoblog.Info("default", "name", r.Name)

	// TODO(user): fill in your defaulting logic.
	if r.Spec.ConcurrencyPolicy == "" {
		r.Spec.ConcurrencyPolicy = AllowConcurrent
	}
	if r.Spec.Suspend == nil {
		// optional must be pointer
		r.Spec.Suspend = new(bool)
	}
	if r.Spec.SuccessfulJobsHistoryLimit == nil {
		// optional must be pointer
		r.Spec.SuccessfulJobsHistoryLimit = new(int32)
		*r.Spec.SuccessfulJobsHistoryLimit = 3
	}
	if r.Spec.FailedJobsHistoryLimit == nil {
		// optional must be pointer
		r.Spec.FailedJobsHistoryLimit = new(int32)
		*r.Spec.FailedJobsHistoryLimit = 1
	}
}
```

marker for validation

```go
// +kubebuilder:webhook:
verbs=create;update;delete,  // ++delete
mutating=false,              // 标识这不是 mutating webhook
path=/validate-batch-tutorial-kubebuilder-io-v1-cronjob,
failurePolicy=fail,
groups=batch.tutorial.kubebuilder.io,resources=cronjobs,versions=v1,
name=vcronjob.kb.io,
sideEffects=None,
admissionReviewVersions=v1
```

对于验证，`ValidateCreate`, `ValidateUpdate` and `ValidateDelete` 方法分别针对 creation/update/deletion 进行验证。

```go
var _ webhook.Validator = &CronJob{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *CronJob) ValidateCreate() (admission.Warnings, error) {
	cronjoblog.Info("validate create", "name", r.Name)

	// TODO(user): fill in your validation logic upon object creation.
	return nil, r.validateCronJob()
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *CronJob) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
	cronjoblog.Info("validate update", "name", r.Name)

	// TODO(user): fill in your validation logic upon object update.
	return nil, r.validateCronJob()
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *CronJob) ValidateDelete() (admission.Warnings, error) {
	cronjoblog.Info("validate delete", "name", r.Name)

	// TODO(user): fill in your validation logic upon object deletion.
	return nil, nil
}
```

分别验证 Name & Spec

```go
func (r *CronJob) validateCronJob() error {
	var allErrs field.ErrorList
	if err := r.validateCronJobName(); err != nil {
		allErrs = append(allErrs, err)
	}
	if err := r.validateCronJobSpec(); err != nil {
		allErrs = append(allErrs, err)
	}
	if len(allErrs) == 0 {
		return nil
	}
	return apierrors.NewInvalid(
		schema.GroupKind{Group: "batch.tutorial.kubebuilder.io", Kind: "CronJob"},
		r.Name, allErrs)
}
```

```go
func (r *CronJob) validateCronJobSpec() *field.Error {
	return validateScheduleFormat(r.Spec.Schedule, field.NewPath("spec").Child("schedule"))
}

func validateScheduleFormat(schedule string, fldPath *field.Path) *field.Error {
	if _, err := cron.ParseStandard(schedule); err != nil {
		return field.Invalid(fldPath, schedule, err.Error())
	}
	return nil
}

func (r *CronJob) validateCronJobName() *field.Error {
	if len(r.ObjectMeta.Name) > validationutils.DNS1035LabelMaxLength-11 {
		return field.Invalid(field.NewPath("metadata").Child("name"), r.Name, "must be no more than 52 characters")
	}
	return nil
}
```

### 9. Running and deploying the controller

Create & Install CRD

```bash
$ make manifests
$ make install
```

```bash
# in case
# The CRD "cronjobs.batch.tutorial.kubebuilder.io" is invalid: 
# metadata.annotations: Too long: must have at most 262144 bytes.
$ kubectl apply -f config/crd/bases/batch.tutorial.kubebuilder.io_cronjobs.yaml --server-side
```

Run controller in foreground

```bash
$ export ENABLE_WEBHOOKS=false
$ make run
```

Create sample CR to test

```yaml
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: CronJob
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: cronjob-sample
spec:
  # TODO(user): Add fields here
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Allow # explicitly specify, but Allow is also default.
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
  
```

```bash
$ kubectl apply -f config/samples/batch_v1_cronjob.yaml
```

Check

```bash
$ kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml
$ kubectl get job
```

**Deploying cert-manager**

[cert-manager](https://github.com/cert-manager/cert-manager) for provisioning the certificates for the webhook server.

CA Injector for injecting the CA bundle into `[Murtating|Validating]WebhookConfiguration`

```bash
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```

Annotation `cert-manager.io/inject-ca-from`, value points to an existing [certificate request instance](https://cert-manager.io/docs/concepts/certificaterequest/) in the format of `<certificate-namespace>/<certificate-name>`

```yaml
# This patch add annotation to admission webhook config and
# CERTIFICATE_NAMESPACE and CERTIFICATE_NAME will be substituted by kustomize
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: mutating-webhook-configuration
  annotations:
    cert-manager.io/inject-ca-from: CERTIFICATE_NAMESPACE/CERTIFICATE_NAME
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/name: validatingwebhookconfiguration
    app.kubernetes.io/instance: validating-webhook-configuration
    app.kubernetes.io/component: webhook
    app.kubernetes.io/created-by: project
    app.kubernetes.io/part-of: project
    app.kubernetes.io/managed-by: kustomize
  name: validating-webhook-configuration
  annotations:
    cert-manager.io/inject-ca-from: CERTIFICATE_NAMESPACE/CERTIFICATE_NAME

```

**Deploying webhooks**

推荐部署一个专门的 [kind](https://book.kubebuilder.io/reference/kind) cluster 用于快速迭代 :smile: 迅速拉起销毁，无需将 webhook 镜像推送至远端。

```bash
# build image locally
$ make docker-build docker-push IMG=<some-registry>/<project-name>:tag
# load local image to kind cluster directly
$ kind load docker-image <your-image-name>:tag --name <your-kind-cluster-name>
```

Enable webhook & cert manager conf through kustomize

- `config/default/kustomization.yaml`
- `config/crd/kustomization.yaml`

```bash
$ make docker-build docker-push IMG=yukanyan/kubebuilder_cronjob:v1.1
$ make deploy IMG=yukanyan/kubebuilder_cronjob:v1.1
```

### 10. Writing tests

