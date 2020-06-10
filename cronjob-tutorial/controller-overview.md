# 控制器简介

控制器是 Kubernetes 的核心，也是任何 operator 的核心。 

控制器的工作是确保对于任何给定的对象，世界的实际状态（包括集群状态，以及潜在的外部状态，如 Kubelet 的运行容器或云提供商的负载均衡器）与对象中的期望状态相匹配。每个控制器专注于一个根Kind，但可能会与其他Kind交互。

我们把这个过程称为 **reconciling**。

在 controller-runtime 中，为特定种类实现 reconciling 的逻辑被称为[* Reconciler *](https://godoc.org/sigs.k8s.io/controller-runtime/pkg/reconcile)。 Reconciler 接受一个对象的名称，并返回我们是否需要再次尝试（例如在错误或周期性控制器的情况下，如 HorizontalPodAutoscaler）。

> 浅析 [emptycontroller.go](./testdata/emptycontroller.go)

首先，我们从一些标准的 import 开始。和之前一样，我们需要核心 controller-runtime 运行库，以及 client 包和我们的 API 类型包。

```
package controllers

import (
    "context"

    "github.com/go-logr/logr"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

接下来，kubebuilder 为我们搭建了一个基本的 reconciler 结构。几乎每一个调节器都需要记录日志，并且能够获取对象，所以可以直接使用。

```
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
}
```

`Reconcile` 实际上是对单个对象进行对账。我们的 Request 只是有一个名字，但我们可以使用 client 从缓存中获取这个对象。

我们返回一个空的结果，没有错误，这就向 controller-runtime 表明我们已经成功地对这个对象进行了对账，在有一些变化之前不需要再尝试对账。

大多数控制器需要一个日志句柄和一个上下文，所以我们在 Reconcile 中将他们初始化。

上下文是用来允许取消请求的，也或者是实现 tracing 等功能。它是所有 client 方法的第一个参数。`Background` 上下文只是一个基本的上下文，没有任何额外的数据或超时时间限制。

控制器-runtime通过一个名为logr的库使用结构化的日志记录。正如我们稍后将看到的，日志记录的工作原理是将键值对附加到静态消息中。我们可以在我们的调和方法的顶部预先分配一些对，让这些对附加到这个调和器的所有日志行。

controller-runtime 通过一个名为 `logr` 日志库使用结构化的记录日志。正如我们稍后将看到的，日志记录的工作原理是将键值对附加到静态消息中。我们可以在我们的 Reconcile 方法的顶部预先分配一些配对信息，将他们加入这个 Reconcile 的所有日志中。

```
func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
    _ = context.Background()
    _ = r.Log.WithValues("cronjob", req.NamespacedName)

    // your logic here

    return ctrl.Result{}, nil
}
```

最后，我们将 Reconcile 添加到 manager 中，这样当 manager 启动时它就会被启动。

现在，我们只是注意到这个 Reconcile 是在 CronJobs 上运行的。以后，我们也会用这个来标记其他的对象。

```
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}
```

现在我们已经了解了 Reconcile 的基本结构，我们来补充一下 `CronJob`s 的逻辑。
