# 实现控制器

CronJob 控制器的基本逻辑如下：

1. 按名称加载 CronJob
2. 列出所有活跃 Job，并更新状态
3. 根据历史限制清理旧 Job
4. 检查控制器本身是否被停止（如果被停止就不要做其他事情了
5. 获取下一次需要运行的 Job
6. 若到达预定时间，没有超过截止日期，并且没有被并发策略所阻止，那么就运行一个新的 Job
7. 当我们看到一个正在运行的作业（自动完成）或者是到了下一个计划运行的时间，将其重新加入队列

> 浅析 [cronjob_controller.go](./testdata/project/controllers/cronjob_controller.go)

我们先从 `import` 开始。下面你会看到，比起那些脚手架为我们生成的 `import`，我们还需要额外 `import` 其他库，我们会在使用的时候逐一讲解。

```
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/go-logr/logr"
    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batch "tutorial.kubebuilder.io/project/api/v1"
)
```

接下来，我们将需在 CronJobReconciler 类型中定义一个时钟，这将允许我们在测试中伪造时间。

```
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    Clock
}
```

我们对时钟进行插桩，以便在测试时更方便地跳转和伪造时间，"真正"的时钟只是调用`time.Now`。

```
type realClock struct{}

func (_ realClock) Now() time.Time { return time.Now() }

// clock knows how to get the current time.
// It can be used to fake out timing for testing.
type Clock interface {
    Now() time.Time
}
```

注意：因为我们现在需要创建和管理 Job，所以我们需要更多的RBAC权限，这意味着要增加几个标记。

```
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete //新增
// +kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get //新增

```

现在，我们进入控制器的核心 -- reconciler 逻辑。

```
var (
    scheduledTimeAnnotation = "batch.tutorial.kubebuilder.io/scheduled-at"
)

func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
    ctx := context.Background()
    log := r.Log.WithValues("cronjob", req.NamespacedName)
```

## 1: 按名称加载 CronJob

我们将使用 client 库来获取 CronJob。所有的 client 方法都以一个上下文（上下文允许停止操作）作为第一个参数，以相关对象作为最后一个参数。Get方法有点特殊，因为它使用一个 NamespacedName 作为中间参数（大多数方法没有中间参数，我们将在下面看到）。

许多 client 方法也会在最后取变量选项。

```
    var cronJob batch.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        // we'll ignore not-found errors, since they can't be fixed by an immediate
        // requeue (we'll need to wait for a new notification), and we can get them
        // on deleted requests.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
```

## 2: 列出所有活跃 Job，并更新状态

为了完全更新我们的状态，我们需要列出这个命名空间中属于这个 CronJob 的所有子 Job。与Get类似，我们可以使用 List 方法来列出子 Job。注意，我们使用变量选项来设置命名空间和字段匹配（它实际上是我们在后面配置的索引检索）。

```
    var childJobs kbatch.JobList
    if err := r.List(ctx, &childJobs, client.InNamespace(req.Namespace), client.MatchingFields{jobOwnerKey: req.Name}); err != nil {
        log.Error(err, "unable to list child Jobs")
        return ctrl.Result{}, err
    }
```

> 这个索引有什么作用？
> The reconciler fetches all jobs owned by the cronjob for the status. As our number of cronjobs increases, looking these up can become quite slow as we have to filter through all of them. For a more efficient lookup, these jobs will be indexed locally on the controller's name. A jobOwnerKey field is added to the cached job objects. This key references the owning controller and functions as the index. Later in this document we will configure the manager to actually index this field.
> 调解器获取cronjob拥有的所有作业的状态。随着我们的cronjob数量的增加，查找这些工作会变得相当慢，因为我们必须过滤掉所有的cronjob。为了更高效的查找，这些作业将在控制器的名称上进行本地索引。一个 jobOwnerKey 字段被添加到缓存的作业对象中。这个键引用了拥有的控制器，并作为索引的功能。在本文档的后面，我们将配置管理器来实际索引这个字段。

Once we have all the jobs we own, we’ll split them into active, successful, and failed jobs, keeping track of the most recent run so that we can record it in status. Remember, status should be able to be reconstituted from the state of the world, so it’s generally not a good idea to read from the status of the root object. Instead, you should reconstruct it every run. That’s what we’ll do here.

一旦我们拥有了所有的作业，我们将把它们分成活动的、成功的和失败的作业，跟踪最近运行的作业，这样我们就可以把它记录在状态中。请记住，状态应该能够从世界的状态中重建，所以一般来说，从根对象的状态中读取不是一个好主意。相反，你应该在每次运行时重新构造它。这就是我们在这里要做的。

We can check if a job is “finished” and whether it succeeded or failed using status conditions. We’ll put that logic in a helper to make our code cleaner.

我们可以使用状态条件来检查一个作业是否 "完成 "以及它是成功还是失败。我们会把这个逻辑放在一个辅助程序中，以使我们的代码更简洁。

```
    // find the active list of jobs
    var activeJobs []*kbatch.Job
    var successfulJobs []*kbatch.Job
    var failedJobs []*kbatch.Job
    var mostRecentTime *time.Time // find the last run so we can update the status
```

We consider a job “finished” if it has a “succeeded” or “failed” condition marked as true. Status conditions allow us to add extensible status information to our objects that other humans and controllers can examine to check things like completion and health.

如果一个作业的 "成功 "或 "失败 "条件被标记为真，我们就认为它 "完成 "了。状态条件允许我们将可扩展的状态信息添加到我们的对象中，其他人类和控制器可以检查诸如完成和健康状况。

```
    isJobFinished := func(job *kbatch.Job) (bool, kbatch.JobConditionType) {
        for _, c := range job.Status.Conditions {
            if (c.Type == kbatch.JobComplete || c.Type == kbatch.JobFailed) && c.Status == corev1.ConditionTrue {
                return true, c.Type
            }
        }

        return false, ""
    }
```

We’ll use a helper to extract the scheduled time from the annotation that we added during job creation.

我们将使用一个助手从我们在创建作业时添加的注解中提取计划时间。

```
    getScheduledTimeForJob := func(job *kbatch.Job) (*time.Time, error) {
        timeRaw := job.Annotations[scheduledTimeAnnotation]
        if len(timeRaw) == 0 {
            return nil, nil
        }

        timeParsed, err := time.Parse(time.RFC3339, timeRaw)
        if err != nil {
            return nil, err
        }
        return &timeParsed, nil
    }
    for i, job := range childJobs.Items {
        _, finishedType := isJobFinished(&job)
        switch finishedType {
        case "": // ongoing
            activeJobs = append(activeJobs, &childJobs.Items[i])
        case kbatch.JobFailed:
            failedJobs = append(failedJobs, &childJobs.Items[i])
        case kbatch.JobComplete:
            successfulJobs = append(successfulJobs, &childJobs.Items[i])
        }

        // We'll store the launch time in an annotation, so we'll reconstitute that from
        // the active jobs themselves.
        scheduledTimeForJob, err := getScheduledTimeForJob(&job)
        if err != nil {
            log.Error(err, "unable to parse schedule time for child job", "job", &job)
            continue
        }
        if scheduledTimeForJob != nil {
            if mostRecentTime == nil {
                mostRecentTime = scheduledTimeForJob
            } else if mostRecentTime.Before(*scheduledTimeForJob) {
                mostRecentTime = scheduledTimeForJob
            }
        }
    }

    if mostRecentTime != nil {
        cronJob.Status.LastScheduleTime = &metav1.Time{Time: *mostRecentTime}
    } else {
        cronJob.Status.LastScheduleTime = nil
    }
    cronJob.Status.Active = nil
    for _, activeJob := range activeJobs {
        jobRef, err := ref.GetReference(r.Scheme, activeJob)
        if err != nil {
            log.Error(err, "unable to make reference to active job", "job", activeJob)
            continue
        }
        cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
    }
```

Here, we’ll log how many jobs we observed at a slightly higher logging level, for debugging. Notice how instead of using a format string, we use a fixed message, and attach key-value pairs with the extra information. This makes it easier to filter and query log lines.

在这里，我们将以稍高的日志级别记录我们观察到的作业数量，以便进行调试。请注意，我们没有使用格式字符串，而是使用固定的消息，并附加键值对与额外信息。这使得过滤和查询日志行更加容易。

```
    log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs", len(failedJobs))
```

Using the date we’ve gathered, we’ll update the status of our CRD. Just like before, we use our client. To specifically update the status subresource, we’ll use the Status part of the client, with the Update method.

使用我们收集到的日期，我们将更新CRD的状态。就像之前一样，我们使用我们的客户端。为了具体更新状态子资源，我们将使用客户端的状态部分，并使用更新方法。

The status subresource ignores changes to spec, so it’s less likely to conflict with any other updates, and can have separate permissions.

状态子资源会忽略规格的变化，所以它不太可能与任何其他更新发生冲突，并且可以有单独的权限。

```
    if err := r.Status().Update(ctx, &cronJob); err != nil {
        log.Error(err, "unable to update CronJob status")
        return ctrl.Result{}, err
    }
```

Once we’ve updated our status, we can move on to ensuring that the status of the world matches what we want in our spec.

一旦我们更新了我们的状态，我们就可以继续确保世界的状态与我们的规范中所希望的一致。

## 3: 根据历史限制清理旧 Job

First, we’ll try to clean up old jobs, so that we don’t leave too many lying around.

首先，我们会尽量清理旧的工作，以免留下太多闲置的工作。

```
    // NB: deleting these is "best effort" -- if we fail on a particular one,
    // we won't requeue just to finish the deleting.
    if cronJob.Spec.FailedJobsHistoryLimit != nil {
        sort.Slice(failedJobs, func(i, j int) bool {
            if failedJobs[i].Status.StartTime == nil {
                return failedJobs[j].Status.StartTime != nil
            }
            return failedJobs[i].Status.StartTime.Before(failedJobs[j].Status.StartTime)
        })
        for i, job := range failedJobs {
            if int32(i) >= int32(len(failedJobs))-*cronJob.Spec.FailedJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete old failed job", "job", job)
            } else {
                log.V(0).Info("deleted old failed job", "job", job)
            }
        }
    }

    if cronJob.Spec.SuccessfulJobsHistoryLimit != nil {
        sort.Slice(successfulJobs, func(i, j int) bool {
            if successfulJobs[i].Status.StartTime == nil {
                return successfulJobs[j].Status.StartTime != nil
            }
            return successfulJobs[i].Status.StartTime.Before(successfulJobs[j].Status.StartTime)
        })
        for i, job := range successfulJobs {
            if int32(i) >= int32(len(successfulJobs))-*cronJob.Spec.SuccessfulJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); (err) != nil {
                log.Error(err, "unable to delete old successful job", "job", job)
            } else {
                log.V(0).Info("deleted old successful job", "job", job)
            }
        }
    }
```

## 4: 检查控制器本身是否被停止

If this object is suspended, we don’t want to run any jobs, so we’ll stop now. This is useful if something’s broken with the job we’re running and we want to pause runs to investigate or putz with the cluster, without deleting the object.

如果这个对象被暂停，我们就不想运行任何作业，所以我们现在就停止。如果我们正在运行的作业出了问题，我们想暂停运行以调查或处理集群，而不删除对象，那么这个方法就很有用。

```
    if cronJob.Spec.Suspend != nil && *cronJob.Spec.Suspend {
        log.V(1).Info("cronjob suspended, skipping")
        return ctrl.Result{}, nil
    }
```

## 5: 获取下一次需要运行的 Job

If we’re not paused, we’ll need to calculate the next scheduled run, and whether or not we’ve got a run that we haven’t processed yet.

We’ll calculate the next scheduled time using our helpful cron library. We’ll start calculating appropriate times from our last run, or the creation of the CronJob if we can’t find a last run.

If there are too many missed runs and we don’t have any deadlines set, we’ll bail so that we don’t cause issues on controller restarts or wedges.

Otherwise, we’ll just return the missed runs (of which we’ll just use the latest), and the next run, so that we can know when it’s time to reconcile again.

如果我们没有暂停，我们就需要计算下一个计划运行的时间，以及我们是否有一个尚未处理的运行。

我们将使用我们有用的cron库计算下一个计划时间。我们将从上一次运行开始计算合适的时间，如果找不到上一次运行，则从CronJob的创建开始计算。

如果错过的运行次数太多，而我们又没有设定任何截止时间，我们就会放弃，这样就不会在控制器重启或楔子上造成问题。

否则，我们只会返回遗漏的运行（其中我们只用最新的），以及下一次运行，这样我们就可以知道什么时候该再次对账。

```
    getNextSchedule := func(cronJob *batch.CronJob, now time.Time) (lastMissed time.Time, next time.Time, err error) {
        sched, err := cron.ParseStandard(cronJob.Spec.Schedule)
        if err != nil {
            return time.Time{}, time.Time{}, fmt.Errorf("Unparseable schedule %q: %v", cronJob.Spec.Schedule, err)
        }

        // for optimization purposes, cheat a bit and start from our last observed run time
        // we could reconstitute this here, but there's not much point, since we've
        // just updated it.
        var earliestTime time.Time
        if cronJob.Status.LastScheduleTime != nil {
            earliestTime = cronJob.Status.LastScheduleTime.Time
        } else {
            earliestTime = cronJob.ObjectMeta.CreationTimestamp.Time
        }
        if cronJob.Spec.StartingDeadlineSeconds != nil {
            // controller is not going to schedule anything below this point
            schedulingDeadline := now.Add(-time.Second * time.Duration(*cronJob.Spec.StartingDeadlineSeconds))

            if schedulingDeadline.After(earliestTime) {
                earliestTime = schedulingDeadline
            }
        }
        if earliestTime.After(now) {
            return time.Time{}, sched.Next(now), nil
        }

        starts := 0
        for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
            lastMissed = t
            // An object might miss several starts. For example, if
            // controller gets wedged on Friday at 5:01pm when everyone has
            // gone home, and someone comes in on Tuesday AM and discovers
            // the problem and restarts the controller, then all the hourly
            // jobs, more than 80 of them for one hourly scheduledJob, should
            // all start running with no further intervention (if the scheduledJob
            // allows concurrency and late starts).
            //
            // However, if there is a bug somewhere, or incorrect clock
            // on controller's server or apiservers (for setting creationTimestamp)
            // then there could be so many missed start times (it could be off
            // by decades or more), that it would eat up all the CPU and memory
            // of this controller. In that case, we want to not try to list
            // all the missed start times.
            starts++
            if starts > 100 {
                // We can't get the most recent times so just return an empty slice
                return time.Time{}, time.Time{}, fmt.Errorf("Too many missed start times (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.")
            }
        }
        return lastMissed, sched.Next(now), nil
    }
    // figure out the next times that we need to create
    // jobs at (or anything we missed).
    missedRun, nextRun, err := getNextSchedule(&cronJob, r.Now())
    if err != nil {
        log.Error(err, "unable to figure out CronJob schedule")
        // we don't really care about requeuing until we get an update that
        // fixes the schedule, so don't return an error
        return ctrl.Result{}, nil
    }
```

We’ll prep our eventual request to requeue until the next job, and then figure out if we actually need to run.

我们将准备我们最终的请求重新排队，直到下一个作业，然后弄清楚我们是否真的需要运行。

```
    scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())} // save this so we can re-use it elsewhere
    log = log.WithValues("now", r.Now(), "next run", nextRun)
```

## 6: Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy
## 6: 若到达预定时间，没有超过截止日期，并且没有被并发策略所阻止，那么就运行一个新的 Job

If we’ve missed a run, and we’re still within the deadline to start it, we’ll need to run a job.

如果我们错过了一次运行，而我们还在最后期限内开始运行，我们就需要运行一次作业。

```
    if missedRun.IsZero() {
        log.V(1).Info("no upcoming scheduled times, sleeping until next")
        return scheduledResult, nil
    }

    // make sure we're not too late to start the run
    log = log.WithValues("current run", missedRun)
    tooLate := false
    if cronJob.Spec.StartingDeadlineSeconds != nil {
        tooLate = missedRun.Add(time.Duration(*cronJob.Spec.StartingDeadlineSeconds) * time.Second).Before(r.Now())
    }
    if tooLate {
        log.V(1).Info("missed starting deadline for last run, sleeping till next")
        // TODO(directxman12): events
        return scheduledResult, nil
    }
```

If we actually have to run a job, we’ll need to either wait till existing ones finish, replace the existing ones, or just add new ones. If our information is out of date due to cache delay, we’ll get a requeue when we get up-to-date information.

如果我们真的要运行一个作业，我们需要等待现有的作业完成，替换现有的作业，或者直接添加新的作业。如果我们的信息由于缓存延迟而过时，当我们得到最新的信息时，我们会得到一个requeue。

```
    // figure out how to run this job -- concurrency policy might forbid us from running
    // multiple at the same time...
    if cronJob.Spec.ConcurrencyPolicy == batch.ForbidConcurrent && len(activeJobs) > 0 {
        log.V(1).Info("concurrency policy blocks concurrent runs, skipping", "num active", len(activeJobs))
        return scheduledResult, nil
    }

    // ...or instruct us to replace existing ones...
    if cronJob.Spec.ConcurrencyPolicy == batch.ReplaceConcurrent {
        for _, activeJob := range activeJobs {
            // we don't care if the job was already deleted
            if err := r.Delete(ctx, activeJob, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete active job", "job", activeJob)
                return ctrl.Result{}, err
            }
        }
    }
```

Once we’ve figured out what to do with existing jobs, we’ll actually create our desired job

We need to construct a job based on our CronJob’s template. We’ll copy over the spec from the template and copy some basic object meta.

Then, we’ll set the “scheduled time” annotation so that we can reconstitute our LastScheduleTime field each reconcile.

Finally, we’ll need to set an owner reference. This allows the Kubernetes garbage collector to clean up jobs when we delete the CronJob, and allows controller-runtime to figure out which cronjob needs to be reconciled when a given job changes (is added, deleted, completes, etc).

一旦我们想好了如何处理现有的工作，我们就会实际创建我们所需的工作。

我们需要基于CronJob的模板来构造一个作业。我们将从模板中复制过来的规范，并复制一些基本的对象元。

然后，我们将设置 "计划时间 "注解，以便我们可以在每次对账时重新构造LastScheduleTime字段。

最后，我们需要设置一个所有者引用。这允许Kubernetes垃圾收集器在我们删除CronJob时清理作业，并允许controller-runtime在给定作业发生变化（被添加、删除、完成等）时，找出哪个cronjob需要进行调节。

```
    constructJobForCronJob := func(cronJob *batch.CronJob, scheduledTime time.Time) (*kbatch.Job, error) {
        // We want job names for a given nominal start time to have a deterministic name to avoid the same job being created twice
        name := fmt.Sprintf("%s-%d", cronJob.Name, scheduledTime.Unix())

        job := &kbatch.Job{
            ObjectMeta: metav1.ObjectMeta{
                Labels:      make(map[string]string),
                Annotations: make(map[string]string),
                Name:        name,
                Namespace:   cronJob.Namespace,
            },
            Spec: *cronJob.Spec.JobTemplate.Spec.DeepCopy(),
        }
        for k, v := range cronJob.Spec.JobTemplate.Annotations {
            job.Annotations[k] = v
        }
        job.Annotations[scheduledTimeAnnotation] = scheduledTime.Format(time.RFC3339)
        for k, v := range cronJob.Spec.JobTemplate.Labels {
            job.Labels[k] = v
        }
        if err := ctrl.SetControllerReference(cronJob, job, r.Scheme); err != nil {
            return nil, err
        }

        return job, nil
    }
    // actually make the job...
    job, err := constructJobForCronJob(&cronJob, missedRun)
    if err != nil {
        log.Error(err, "unable to construct job from template")
        // don't bother requeuing until we get a change to the spec
        return scheduledResult, nil
    }

    // ...and create it on the cluster
    if err := r.Create(ctx, job); err != nil {
        log.Error(err, "unable to create Job for CronJob", "job", job)
        return ctrl.Result{}, err
    }

    log.V(1).Info("created Job for CronJob run", "job", job)
```

## 7: 当我们看到一个正在运行的作业（自动完成）或者是到了下一个计划运行的时间，将其重新加入队列

Finally, we’ll return the result that we prepped above, that says we want to requeue when our next run would need to occur. This is taken as a maximum deadline -- if something else changes in between, like our job starts or finishes, we get modified, etc, we might reconcile again sooner.

最后，我们会返回上面我们准备的结果，说我们要重新排队，当我们的下一次运行需要发生。这是作为一个最大的截止日期--如果中间有其他的变化，比如我们的作业开始或结束，我们被修改了等等，我们可能会更快地再次对账。

```
    // we'll requeue once we see the running job, and update our status
    return scheduledResult, nil
}
```

## 安装

Finally, we’ll update our setup. In order to allow our reconciler to quickly look up Jobs by their owner, we’ll need an index. We declare an index key that we can later use with the client as a pseudo-field name, and then describe how to extract the indexed value from the Job object. The indexer will automatically take care of namespaces for us, so we just have to extract the owner name if the Job has a CronJob owner.

Additionally, we’ll inform the manager that this controller owns some Jobs, so that it will automatically call Reconcile on the underlying CronJob when a Job changes, is deleted, etc.

最后，我们会更新我们的设置。为了让我们的对账器能够根据作业的所有者快速查找作业，我们需要一个索引。我们声明一个索引键，以后我们可以用它作为客户端的伪字段名，然后描述如何从Job对象中提取索引值。索引器会自动为我们处理命名空间，所以如果Job有CronJob所有者，我们只需要提取所有者名称即可。

另外，我们会通知管理器这个控制器拥有一些Job，这样当一个Job发生变化、被删除等情况时，它就会自动调用底层CronJob的Reconcile。

```
var (
    jobOwnerKey = ".metadata.controller"
    apiGVStr    = batch.GroupVersion.String()
)

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // set up a real clock, since we're not in a test
    if r.Clock == nil {
        r.Clock = realClock{}
    }

    if err := mgr.GetFieldIndexer().IndexField(&kbatch.Job{}, jobOwnerKey, func(rawObj runtime.Object) []string {
        // grab the job object, extract the owner...
        job := rawObj.(*kbatch.Job)
        owner := metav1.GetControllerOf(job)
        if owner == nil {
            return nil
        }
        // ...make sure it's a CronJob...
        if owner.APIVersion != apiGVStr || owner.Kind != "CronJob" {
            return nil
        }

        // ...and if so, return it
        return []string{owner.Name}
    }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&batch.CronJob{}).
        Owns(&kbatch.Job{}).
        Complete(r)
}
```

That was a doozy, but now we've got a working controller.  Let's test against the cluster, then, if we don't have any issues, deploy it!

这真是个大麻烦，但现在我们已经有了一个工作控制器。 让我们来测试一下 对照集群，然后，如果我们没有任何问题，部署它!