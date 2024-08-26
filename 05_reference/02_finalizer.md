`Finalizers` 允许 controller 实现异步 pre-delete hooks，定义在 `Reconcile` 方法中。

关键：删除先触发了一次更新 = 设置 deletion timestamp。

- 若对象未被删除且未注册 finalizer，则在 Kubernetes 中添加 finalizer 并更新对象。
- 如果对象正在被删除且 finalizer 仍然存在于 finalizers 列表中，则执行预删除逻辑并移除 finalizer 并更新对象。
- 确保删除逻辑**幂等**。

```go
func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := r.Log.WithValues("cronjob", req.NamespacedName)

	cronJob := &batchv1.CronJob{}
	if err := r.Get(ctx, req.NamespacedName, cronJob); err != nil {
		log.Error(err, "unable to fetch CronJob")
		// we'll ignore not-found errors, since they can't be fixed by an immediate
		// requeue (we'll need to wait for a new notification), and we can get them
		// on deleted requests.
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// name of our custom finalizer 自定义 finalizer 名称
	myFinalizerName := "batch.tutorial.kubebuilder.io/finalizer"

	// examine DeletionTimestamp to determine if object is under deletion
    // 检查是否有删除时间戳判断对象是否处于删除中
	if cronJob.ObjectMeta.DeletionTimestamp.IsZero() {
		// The object is not being deleted, so if it does not have our finalizer,
		// then lets add the finalizer and update the object. This is equivalent
		// to registering our finalizer.
        // 对象未被删除，那么必然没有自定义 finalizer，那就添加并更新对象
		if !controllerutil.ContainsFinalizer(cronJob, myFinalizerName) {
			controllerutil.AddFinalizer(cronJob, myFinalizerName)
			if err := r.Update(ctx, cronJob); err != nil {
				return ctrl.Result{}, err
			}
		}
	} else {
		// The object is being deleted
        // 对象删除中
        // 判断是否包含自定义 finalizer
		if controllerutil.ContainsFinalizer(cronJob, myFinalizerName) {
			// our finalizer is present, so lets handle any external dependency
            // 自定义 finalizer 存在，处理外部依赖
			if err := r.deleteExternalResources(cronJob); err != nil {
				// if fail to delete the external dependency here, return with error
				// so that it can be retried.
				return ctrl.Result{}, err
			}

			// remove our finalizer from the list and update it.
            // 外部依赖处理结束，移除自定义 finalizer 并更新对象
			controllerutil.RemoveFinalizer(cronJob, myFinalizerName)
			if err := r.Update(ctx, cronJob); err != nil {
				return ctrl.Result{}, err
			}
		}

		// Stop reconciliation as the item is being deleted
		return ctrl.Result{}, nil
	}

	// Your reconcile logic

	return ctrl.Result{}, nil
}

func (r *Reconciler) deleteExternalResources(cronJob *batch.CronJob) error {
	//
	// delete any external resources associated with the cronJob
	//
	// Ensure that delete implementation is idempotent and safe to invoke
	// multiple times for same object.
}
```

