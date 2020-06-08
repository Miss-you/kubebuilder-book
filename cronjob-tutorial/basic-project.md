# 基础项目里面有什么组件？

在脚手架生成新项目时，Kubebebuilder 为我们提供了一些基本的模板。

## 创建基础设施

首先是基本的项目文件初始化，为项目构建做好准备。

> `go.mod`: 我们的项目的Go mod配置文件，记录依赖库信息。

```go
module tutorial.kubebuilder.io/project

go 1.13

require (
    github.com/go-logr/logr v0.1.0
    github.com/robfig/cron v1.2.0
    k8s.io/api v0.17.2
    k8s.io/apimachinery v0.17.2
    k8s.io/client-go v0.17.2
    sigs.k8s.io/controller-runtime v0.5.0
)
```

> `Makefile`: 用于控制器构建和部署的Makefile文件

```makefile
# Image URL to use all building/pushing image targets
IMG ?= controller:latest
# Produce CRDs that work back to Kubernetes 1.11 (no version conversion)
CRD_OPTIONS ?= "crd:trivialVersions=true"

all: manager

# Run tests
test: generate fmt vet manifests
    go test ./api/... ./controllers/... -coverprofile cover.out

# Build manager binary
manager: generate fmt vet
    go build -o bin/manager main.go

# Run against the configured Kubernetes cluster in ~/.kube/config
run: generate fmt vet
    go run ./main.go

# Install CRDs into a cluster
install: manifests
    kubectl apply -f config/crd/bases

# Deploy controller in the configured Kubernetes cluster in ~/.kube/config
deploy: manifests
    kubectl apply -f config/crd/bases
    kustomize build config/default | kubectl apply -f -

# Generate manifests e.g. CRD, RBAC etc.
manifests: controller-gen
    $(CONTROLLER_GEN) $(CRD_OPTIONS) rbac:roleName=manager-role webhook paths="./api/...;./controllers/..." output:crd:artifacts:config=config/crd/bases

# Run go fmt against code
fmt:
    go fmt ./...

# Run go vet against code
vet:
    go vet ./...

# Generate code
generate: controller-gen
    $(CONTROLLER_GEN) object:headerFile=./hack/boilerplate.go.txt paths=./api/...

# Build the docker image
docker-build: test
    docker build . -t ${IMG}
    @echo "updating kustomize image patch file for manager resource"
    sed -i'' -e 's@image: .*@image: '"${IMG}"'@' ./config/default/manager_image_patch.yaml

# Push the docker image
docker-push:
    docker push ${IMG}

# find or download controller-gen
# download controller-gen if necessary
controller-gen:
ifeq (, $(shell which controller-gen))
    go get sigs.k8s.io/controller-tools/cmd/controller-gen@v0.2.0-rc.0
CONTROLLER_GEN=$(shell go env GOPATH)/bin/controller-gen
else
CONTROLLER_GEN=$(shell which controller-gen)
endif
```

> `PROJECT`: 用于生成组件的 Kubebuilder 元数据

```yaml
version: "2"
domain: tutorial.kubebuilder.io
repo: tutorial.kubebuilder.io/project
```

## 启动配置

我们还可以在[`config/`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config)目录下获得启动配置。现在，它只包含了在集群上启动控制器所需的[Kustomize](https://sigs.k8s.io/kustomize) YAML 定义，但一旦我们开始编写控制器，它还将包含我们的 CustomResourceDefinitions(CRD) 、RBAC 配置和 WebhookConfigurations 。

[`config/default`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/default) 在标准配置中包含 [Kustomize base](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/config/default/kustomization.yaml) ，它用于启动控制器。

其他每个目录都包含一个不同的配置，重构为自己的基础。

- [`config/manager`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/manager): 在集群中以pod的形式启动控制器

- [`config/rbac`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/rbac): 在自己的账户下运行控制器所需的权限

## 入口函数

最后，当然也是最重要的一点，生成项目的入口函数：`main.go`。接下来我们看看它......
