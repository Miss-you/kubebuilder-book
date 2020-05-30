# 快速入门

本快速入门指南将涵盖以下内容：

- [快速入门](#快速入门)
  - [依赖组件](#依赖组件)
  - [安装](#安装)
  - [创建一个项目](#创建一个项目)
  - [创建一个 API](#创建一个-api)
  - [测试](#测试)
  - [安装 CR 实例](#安装-cr-实例)
  - [如何在集群中运行](#如何在集群中运行)
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

<aside class="note">
<h1> 如果你的安装目录不在 `$GOPATH` 中 </h1>

如果你的 kubebuilder 安装目录不在 `$GOPATH` 中，你需要运行 `go mod init <modulename>` 来告诉 kubebuilder 和 Go module 的基本导入路径。

若要进一步了解 `GOPATH`，参阅 [如何编写Go代码][how-to-write-go-code-golang-docs] 页面文档中的 [GOPATH 环境变量][GOPATH-golang-docs] 章节。   
</aside>

<aside class="note">
<h1> Go package 问题 </h1>
确保你已经执行 `$ export GO111MODULE=on` 命令来激活模块支持，以解决像 `cannot find package .... (from $GOROOT)` 这样的问题。
</aside>

## 创建一个 API

Run the following command to create a new API (group/version) as `webapp/v1` and the new Kind(CRD) `Guestbook` on it:

```bash
kubebuilder create api --group webapp --version v1 --kind Guestbook
```

<aside class="note">
<h1>Press Options</h1>

If you press `y` for Create Resource [y/n] and for Create Controller [y/n] then this will create the files `api/v1/guestbook_types.go` where the API is defined 
and the `controller/guestbook_controller.go` where the reconciliation business logic is implemented for this Kind(CRD).

</aside>


**OPTIONAL:** Edit the API definition and the reconciliation business
logic. For more info see [Designing an API](/cronjob-tutorial/api-design.md) and [What's in
a Controller](cronjob-tutorial/controller-overview.md).

<details><summary>Click here to see an example. `(api/v1/guestbook_types.go)` </summary>
<p>

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

</p>
</details>


## 测试

You'll need a Kubernetes cluster to run against.  You can use
[KIND](https://sigs.k8s.io/kind) to get a local cluster for testing, or
run against a remote cluster.

<aside class="note">
<h1>Context Used</h1>

Your controller will automatically use the current context in your
kubeconfig file (i.e. whatever cluster `kubectl cluster-info` shows).

</aside> 

Install the CRDs into the cluster:
```bash
make install
```

Run your controller (this will run in the foreground, so switch to a new
terminal if you want to leave it running):
```bash
make run
```

## 安装 CR 实例

If you pressed `y` for Create Resource [y/n] then you created an (CR)Custom Resource for your (CRD)Custom Resource Definition in your samples (make sure to edit them first if you've changed the
API definition):

```bash
kubectl apply -f config/samples/
```

## 如何在集群中运行

Build and push your image to the location specified by `IMG`:

```bash
make docker-build docker-push IMG=<some-registry>/<project-name>:tag
```

Deploy the controller to the cluster with image specified by `IMG`:

```bash
make deploy IMG=<some-registry>/<project-name>:tag
```

<aside class="note">
<h1>RBAC errors</h1>

If you encounter RBAC errors, you may need to grant yourself cluster-admin
privileges or be logged in as admin. See [Prerequisites for using Kubernetes RBAC on GKE cluster v1.11.x and older][pre-rbc-gke] which may be your case.  

</aside> 

## 卸载 CRD

To delete your CRDs from the cluster:

```bash
make uninstall
```

## 下一步 

Now, follow up the [CronJob tutorial][cronjob-tutorial] to better understand how it works by developing a demo example project. 

[pre-rbc-gke]:https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#iam-rolebinding-bootstrap
[cronjob-tutorial]: https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html
[GOPATH-golang-docs]: https://golang.org/doc/code.html#GOPATH
[how-to-write-go-code-golang-docs]: https://golang.org/doc/code.html 

