# 快速入门

本快速入门指南将涵盖以下内容：

- [快速入门](#快速入门)
  - [依赖组件](#依赖组件)
  - [安装](#安装)
  - [创建一个项目](#创建一个项目)
      - [如果你的安装目录不在 `$GOPATH` 中](#如果你的安装目录不在-gopath-中)
      - [Go package 问题](#go-package-问题)
  - [创建一个 API](#创建一个-api)
      - [创建选项](#创建选项)
      - [示例](#示例)
  - [测试](#测试)
      - [使用的上下文](#使用的上下文)
  - [安装 CR 实例](#安装-cr-实例)
  - [如何在集群中运行](#如何在集群中运行)
      - [RBAC 错误](#rbac-错误)
  - [卸载 CRD](#卸载-crd)
  - [下一步](#下一步)

## 依赖组件

- [go](https://golang.org/dl/) version v1.13+.
- [docker](https://docs.docker.com/install/) version 17.03+.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.11.3+.
- [kustomize](https://sigs.k8s.io/kustomize/docs/INSTALL.md) v3.1.0+
- 能够访问 Kubernetes v1.11.3+ 集群

## 安装

安装 [kubebuilder](https://sigs.k8s.io/kubebuilder):

```bash
os=$(go env GOOS)
arch=$(go env GOARCH)

# 下载 kubebuilder 并解压到 tmp 目录中
curl -L https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/

# 将 kubebuilder 移动到一个长期的路径，并将其加入环境变量 path 中 
# (如果你把 kubebuilder 放在别的地方，你需要额外设置 KUBEBUILDER_ASSETS 环境变量)
sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```

> 请使用 master 分支的 kubebuilder 进行构建

> 另外，你可以从 `https://go.kubebuilder.io/dl/latest/${os}/${arch}` 下载安装包

## 创建一个项目

创建一个目录，然后在里面运行 `kubebuilder init` 命令，初始化一个新项目。示例如下。

```bash
mkdir $GOPATH/src/example
cd $GOPATH/src/example
kubebuilder init --domain my.domain
```

#### 如果你的安装目录不在 `$GOPATH` 中

如果你的 kubebuilder 安装目录不在 `$GOPATH` 中，你需要运行 `go mod init <modulename>` 来告诉 kubebuilder 和 Go module 的基本导入路径。

若要进一步了解 `GOPATH`，参阅 [如何编写Go代码][how-to-write-go-code-golang-docs] 页面文档中的 [GOPATH 环境变量][GOPATH-golang-docs] 章节。   

#### Go package 问题 

确保你已经执行 `$ export GO111MODULE=on` 命令来激活模块支持，以解决像 `cannot find package .... (from $GOROOT)` 这样的问题。

## 创建一个 API

运行下面的命令，创建一个新的API（组/版本）为 "webapp/v1"，并在上面创建新的 Kind(CRD) "Guestbook"。

```bash
kubebuilder create api --group webapp --version v1 --kind Guestbook
```

#### 创建选项

如果你在 Create Resource [y/n] 和 Create Controller [y/n] 中按`y`，那么这将创建文件 `api/v1/guestbook_types.go` ，该文件中定义相关 API ，而针对于这一类型 (CRD) 的对账业务逻辑生成在 `controller/guestbook_controller.go` 文件中。

**可选项：** 编辑API定义和对账业务逻辑。更多信息请参见[设计一个API](/cronjob-tutorial/api-design.md)和[控制器](cronjob-tutorial/controller-overview.md)。

#### 示例

```go
// GuestbookSpec defines the desired state of Guestbook
type GuestbookSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Quantity of instances
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=10
	Size int32 `json:"size"`

	// Name of the ConfigMap for GuestbookSpec's configuration
	// +kubebuilder:validation:MaxLength=15
	// +kubebuilder:validation:MinLength=1
	ConfigMapName string `json:"configMapName"`

	// +kubebuilder:validation:Enum=Phone;Address;Name
	Type string `json:"alias,omitempty"`
}

// GuestbookStatus defines the observed state of Guestbook
type GuestbookStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// PodName of the active Guestbook node.
	Active string `json:"active"`

	// PodNames of the standby Guestbook nodes.
	Standby []string `json:"standby"`
}

type Guestbook struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GuestbookSpec   `json:"spec,omitempty"`
	Status GuestbookStatus `json:"status,omitempty"`
}
```

## 测试

你需要一个Kubernetes集群来运行。 你可以使用[KIND](https://sigs.k8s.io/kind)来获取一个本地集群进行测试，也可以在远程集群上运行。

#### 使用的上下文

你的控制器将自动使用你的 `kubeconfig` 文件中的当前上下文(即无论群集 `kubectl cluster-info` 显示的是什么群集)。

将CRD安装到集群中

```bash
make install
```

运行控制器（这将在前台运行，如果你想让它一直运行，请切换到新的终端）。

```bash
make run
```

## 安装 CR 实例

如果你按了 `y` 创建资源[y/n]，那么你就为示例中的自定义资源定义 `CRD` 创建了一个自定义资源 `CR` (如果你更改了API定义，请务必先编辑它们)。

```bash
kubectl apply -f config/samples/
```

## 如何在集群中运行

构建并推送你的镜像到 `IMG` 指定的位置。

```bash
make docker-build docker-push IMG=<some-registry>/<project-name>:tag
```

根据 `IMG` 指定的镜像将控制器部署到集群中。


```bash
make deploy IMG=<some-registry>/<project-name>:tag
```

#### RBAC 错误

如果你遇到RBAC错误，你可能需要授予自己集群管理员权限或以管理员身份登录。请参考[在GKE集群 v1.11.x 及以上版本上使用 Kubernetes RBAC 的组件依赖][pre-rbc-gke]可能是你的情况。 

## 卸载 CRD

从你的集群中删除 CRD

```bash
make uninstall
```

## 下一步 

现在，参照[CronJob教程][cronjob-tutorial]，通过开发一个演示示例项目更好地理解 kubebuilder 的工作原理。

[pre-rbc-gke]:https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#iam-rolebinding-bootstrap
[cronjob-tutorial]: https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html
[GOPATH-golang-docs]: https://golang.org/doc/code.html#GOPATH
[how-to-write-go-code-golang-docs]: https://golang.org/doc/code.html 

