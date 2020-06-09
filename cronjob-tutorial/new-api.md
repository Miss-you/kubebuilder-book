# 添加一个 API

我们可以使用 `kubebuilder create api` 命令创建一个新的 Kind (你注意到了[上一章](./gvks.md#kinds-and-resources)的内容，对吧？) 和相应的控制器。

```bash
kubebuilder create api --group batch --version v1 --kind CronJob
```

当我们第一次为一个组版本调用这个命令时，它将为新的组版本创建一个目录。

在这种情况下，[`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) 目录被创建了，对应于 `batch.tutorial.kubebuilder.io/v1` (还记得我们从一开始指定的 [`--domain` 配置](cronjob-tutorial.md#scaffolding-out-our-project) 吗？)

它还为我们的 `CronJob` 类型添加了一个文件：`api/v1/cronjob_types.go`。每次我们对不同的 Kind 调用该命令时，它都会添加一个相应的新文件。
 
下面我们就来了解一下，先来看看都有哪些内容，然后就可以针对每部分讲解代码。

浅析[emptyapi.go](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/emptyapi.go)

我们开始的时候就足够简单：我们导入meta/v1 API组，它本身通常不会公开，而是包含了所有Kubernetes Kinds通用的元数据。

文件的开头比较简单：我们 import `meta/v1` API组，它包含了所有 Kubernetes Kinds 共有的元数据。

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

接下来，我们为 Kind 的 Spec 和 Status 定义类型。Kubernetes 通过将期望状态（Spec）与实际集群状态（其他对象的Status）和外部状态进行对账，然后记录下它所观察到的对象（Status）来实现功能。因此，每个功能对象都包括 Spec 和 Status。当然也有例外，部分类型，比如 `ConfigMap` 并不遵循这种模式，因为它们不编码期望状态，除此之外大多数类型都是这种模式。

```go
// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// CronJobSpec 定义了 CronJob 的期望状态
type CronJobSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}

// CronJobStatus 定义了 CronJob 的实际状态
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}
```

接下来，我们定义与实际 Kinds 对应的类型，`CronJob` 和 `CronJobList`。`CronJob` 是我们的根类型，描述了 `CronJob` 类型。和所有 Kubernetes 对象一样，它包含了 `TypeMeta`（描述 API 版本和 Kind），还包含了 `ObjectMeta`，它保存了名称、命名空间和标签等东西。

`CronJobList` 只是包含多个 `CronJob`s 的容器。它在批量操作中使用，比如 LIST 或者 MAP。

一般情况下，我们不会修改这俩对象中任何一个----所有的修改都在 Spec 或 Status 中。

`+kubebuilder:object:root` 注释叫做标记。我们稍后会看到更多的标记，但要知道它们作为额外的元数据，它会告诉 [controller-tools](https://github.com/kubernetes-sigs/controller-tools)（我们的代码和 YAML 生成器）额外的信息。这个特殊的类型告诉 `object` 生成器这个类型代表一个 Kind。然后，`object` 生成器为我们生成一个 [runtime.Object](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Object) 接口，该接口是所有 `Kind`s 类型必须实现的标准接口。

```go
// +kubebuilder:object:root=true

// CronJob is the Schema for the cronjobs API
type CronJob struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}
```

最后，我们将 Go 类型添加到 API 组中。这样我们就可以将这个 API 组中的类型添加到任何 [Scheme](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Scheme)。

```go
func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

既然看完了生成 api 代码的基本结构，那我们就来补上相关类型定义吧！
