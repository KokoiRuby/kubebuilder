API version changes release to release.

### Changing things up

在 K8s 中，所有版本之间必须能够安全地往返转换 = 从 v1 → v2 再回退到 v1，要求不能丢失任何信息。

任何对 API 的修改都必须确保前向兼容性。

```yaml
# v1
schedule: "*/1 * * * *"

# v2
schedule:
  minute: */1
```

```bash
# Press y for “Create Resource” and n for “Create Controller”
$ kubebuilder create api \
	--group batch \
	--version v2 \
	--kind CronJob
```

```go
package v2

type CronJobSpec struct {
	// +kubebuilder:validation:MinLength=0

	// The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
	// Schedule string `json:"schedule"`
	Schedule CronSchedule `json:"schedule"`
  	...
}

type CronSchedule struct {
	// specifies the minute during which the job executes.
	// +optional
	Minute *CronField `json:"minute,omitempty"`
	// specifies the hour during which the job executes.
	// +optional
	Hour *CronField `json:"hour,omitempty"`
	// specifies the day of the month during which the job executes.
	// +optional
	DayOfMonth *CronField `json:"dayOfMonth,omitempty"`
	// specifies the month during which the job executes.
	// +optional
	Month *CronField `json:"month,omitempty"`
	// specifies the day of the week during which the job executes.
	// +optional
	DayOfWeek *CronField `json:"dayOfWeek,omitempty"`
}

// represents a Cron field specifier.
type CronField string
```

**Storage Version** 对于多版本，必须指定一个存储的版本，这里选择 v1 进行存储。

```go
package v1

// +kubebuilder:storageversion
type CronJob struct {
    ...
}
```

### 2. Hubs, spokes, and other wheel metaphors

对于用户而言，可以请求不同版本 API。对于 CRD，**通过 webhook 实现版本转换**，类似 defaulting & validating

版本数较少时，可以通过转换函数 :cry: 一旦数量上去...

**“hub and spoke” 模型**：标记一个版本为 hub，其他所有版本定义 to/from 的转换函数

当我们想要转换 2 个非 hub 版本时，利用 hub 作一个跳板即可。

```bash
#
#     →  ConvertTo()  →   → ConvertFrom() →
# v2                   v1                   v3
#     ← ConvertFrom() ←   ←  ConvertTo()  ←
# 
```

**Webhook 的作用**：当客户端请求某个版本时，可能和存储在 API Server 内的版本不一致，∵ 需要调用一个 webhook 进行转换。

controller-runtime 实现该 webook 以及负责 hub-and-spoke 转换。

### 3. Implementing conversion

Hub `project/api/v1/cronjob_conversion.go`

```go
package v1

// mark as conversion hub
func (*CronJob) Hub() {}
```

Spokes `project/api/v2/cronjob_conversion.go`

```go
package v2

// to hub v1
func (src *CronJob) ConvertTo(dstRaw conversion.Hub) error {
	// type conversion
	dst := dstRaw.(*v1.CronJob)

	sched := src.Spec.Schedule
	scheduleParts := []string{"*", "*", "*", "*", "*"}

	if sched.Minute != nil {
		scheduleParts[0] = string(*sched.Minute)
	}
	if sched.Hour != nil {
		scheduleParts[1] = string(*sched.Hour)
	}
	if sched.DayOfMonth != nil {
		scheduleParts[2] = string(*sched.DayOfMonth)
	}
	if sched.Month != nil {
		scheduleParts[3] = string(*sched.Month)
	}
	if sched.DayOfWeek != nil {
		scheduleParts[4] = string(*sched.DayOfWeek)
	}
	// "* * * * *"
	dst.Spec.Schedule = strings.Join(scheduleParts, " ")

	// ObjectMeta
	dst.ObjectMeta = src.ObjectMeta

	// Spec
	dst.Spec.StartingDeadlineSeconds = src.Spec.StartingDeadlineSeconds
	dst.Spec.ConcurrencyPolicy = v1.ConcurrencyPolicy(src.Spec.ConcurrencyPolicy)
	dst.Spec.Suspend = src.Spec.Suspend
	dst.Spec.JobTemplate = src.Spec.JobTemplate
	dst.Spec.SuccessfulJobsHistoryLimit = src.Spec.SuccessfulJobsHistoryLimit
	dst.Spec.FailedJobsHistoryLimit = src.Spec.FailedJobsHistoryLimit

	// Status
	dst.Status.Active = src.Status.Active
	dst.Status.LastScheduleTime = src.Status.LastScheduleTime
	return nil

}

// from hub v1
func (dst *CronJob) ConvertFrom(srcRaw conversion.Hub) error {
	// type conversion
	src := srcRaw.(*v1.CronJob)

	// "* * * * *"
	schedParts := strings.Split(src.Spec.Schedule, "")
	if len(schedParts) != 5 {
		return fmt.Errorf("invalid schedule: not a standard 5-field schedule")
	}

	partIfNeeded := func(raw string) *CronField {
		if raw == "*" {
			return nil
		}
		// to string
		part := CronField(raw)
		return &part
	}

	dst.Spec.Schedule.Minute = partIfNeeded(schedParts[0])
	dst.Spec.Schedule.Hour = partIfNeeded(schedParts[1])
	dst.Spec.Schedule.DayOfMonth = partIfNeeded(schedParts[2])
	dst.Spec.Schedule.Month = partIfNeeded(schedParts[3])
	dst.Spec.Schedule.DayOfWeek = partIfNeeded(schedParts[4])

	// ObjectMeta
	dst.ObjectMeta = src.ObjectMeta

	// Spec
	dst.Spec.StartingDeadlineSeconds = src.Spec.StartingDeadlineSeconds
	dst.Spec.ConcurrencyPolicy = ConcurrencyPolicy(src.Spec.ConcurrencyPolicy)
	dst.Spec.Suspend = src.Spec.Suspend
	dst.Spec.JobTemplate = src.Spec.JobTemplate
	dst.Spec.SuccessfulJobsHistoryLimit = src.Spec.SuccessfulJobsHistoryLimit
	dst.Spec.FailedJobsHistoryLimit = src.Spec.FailedJobsHistoryLimit

	// Status
	dst.Status.Active = src.Status.Active
	dst.Status.LastScheduleTime = src.Status.LastScheduleTime

	return nil
}
```

### 4. Setting up the webhooks

```bash
# already did
$ kubebuilder create webhook \
	--group batch \
	--version v1 \
	--kind CronJob \
	--conversion
```

`project/api/v1/cronjob_webhook.go` 只要 types 实现了 Hub & Convertible 接口，conversion webhook 就会被注册到 manager 中。

```go
func (r *CronJob) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(r).
        Complete()
}
```

```bash
# for v2
$ kubebuilder create webhook \
	--group batch \
	--version v2 \
	--kind CronJob \
	--conversion
```

`project/api/v2/cronjob_webhook.go` 实现 Defaulting & Validating

```go
func (r *CronJob) Default() {
	cronjoblog.Info("default", "name", r.Name)
	if r.Spec.ConcurrencyPolicy == "" {
		r.Spec.ConcurrencyPolicy = AllowConcurrent
	}
	if r.Spec.Suspend == nil {
		r.Spec.Suspend = new(bool)
	}
	if r.Spec.SuccessfulJobsHistoryLimit == nil {
		r.Spec.SuccessfulJobsHistoryLimit = new(int32)
		*r.Spec.SuccessfulJobsHistoryLimit = 3
	}
	if r.Spec.FailedJobsHistoryLimit == nil {
		r.Spec.FailedJobsHistoryLimit = new(int32)
		*r.Spec.FailedJobsHistoryLimit = 1
	}
}
```

```go
// diff
func (r *CronJob) validateCronJobSpec() *field.Error {
	// Build cron expression from the parts
	parts := []string{"*", "*", "*", "*", "*"} // default parts for minute, hour, day of month, month, day of week
	if r.Spec.Schedule.Minute != nil {
		parts[0] = string(*r.Spec.Schedule.Minute) // Directly cast CronField (which is an alias of string) to string
	}
	if r.Spec.Schedule.Hour != nil {
		parts[1] = string(*r.Spec.Schedule.Hour)
	}
	if r.Spec.Schedule.DayOfMonth != nil {
		parts[2] = string(*r.Spec.Schedule.DayOfMonth)
	}
	if r.Spec.Schedule.Month != nil {
		parts[3] = string(*r.Spec.Schedule.Month)
	}
	if r.Spec.Schedule.DayOfWeek != nil {
		parts[4] = string(*r.Spec.Schedule.DayOfWeek)
	}

	// Join parts to form the full cron expression
	cronExpression := strings.Join(parts, " ")

	return validateScheduleFormat(
		cronExpression,
		field.NewPath("spec").Child("schedule"))
}
```

`project/cmd/main.go`

```go
if os.Getenv("ENABLE_WEBHOOKS") != "false" {
	if err = (&batchv1.CronJob{}).SetupWebhookWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create webhook", "webhook", "CronJob")
		os.Exit(1)
	}
	// ++
    if err = (&batchv2.CronJob{}).SetupWebhookWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create webhook", "webhook", "CronJob")
		os.Exit(1)
	}
}
```

### 4. Deployment and Testing

`config/crd/kustomization.yaml`

```yaml
# enable patches
patches:
- path: patches/webhook_in_cronjobs.yaml
- path: patches/cainjection_in_cronjobs.yaml
```

`config/default/kustomization.yaml`

```yaml
# enable
resources:
- ../webhook
- ../certmanager

patches:
- path: manager_webhook_patch.yaml
- path: webhookcainjection_patch.yaml
```

Create & Deploy CRD

```bash
$ make manifests
$ make install
```

Deploy controller

```bash
$ make docker-build docker-push IMG=yukanyan/kubebuilder_cronjob:v1.2.1
$ make deploy IMG=yukanyan/kubebuilder_cronjob:v1.2.1
```

**Test** `config/samples`

```yaml
apiVersion: batch.tutorial.kubebuilder.io/v2
kind: CronJob
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: cronjob-sample
spec:
  schedule:
    minute: "*/1"
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
$ kubectl apply -f config/samples/batch_v2_cronjob.yaml
```

Check

```bash
kubectl get cronjobs.v2.batch.tutorial.kubebuilder.io -o yaml
kubectl get cronjobs.v1.batch.tutorial.kubebuilder.io -o yaml
```

