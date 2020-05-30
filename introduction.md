**注：** 没有耐心的读者可直接进入[快速入门](quick-start.md).

**使用Kubebuilder v1? 请参考[v1](https://book-v1.book.kubebuilder.io)**

## 适合读者类型

#### Kubernetes 用户

Kubernetes用户将通过学习如何设计和实现API背后的基本概念，对Kubernetes有更深刻的理解。本书将教会读者如何开发自己的 Kubernetes API，以及 Kubernetes API 的核心设计原则。

内容如下：
- Kubernetes API和资源的结构
- API版本的语义
- 自愈机制
- 垃圾收集和终结器
- 声明式API与强制式API
- 基于水平的API与基于边缘的API
- 资源与子资源

#### Kubernetes API 开发者

API扩展开发者将学习到实现 Kubernetes API 的原理和概念，以及一些用于加快开发的工具和库。本书涵盖了 Kubernetes API 开发者经常遇到的陷阱和误区。

内容如下：
- 如何将多个事件批处理成一个对账（调和）调用
- 如何配置定期对账
- *即将出版*
    - 什么时候使用 lister cache 和 live lookups？
    - 垃圾收集器 VS 终结器
    - 如何使用声明式校验和 Webhook 校验
    - 如何实现 API 版本化

## 贡献

如果您愿意为这本书或代码做贡献，请您务必先耐心阅读我们的[贡献](https://github.com/kubernetes-sigs/kubebuilder/blob/master/CONTRIBUTING.md)指南。

## 资源

* 项目: [sigs.k8s.io/kubebuilder](https://sigs.k8s.io/kubebuilder)
* Slack: [#kubebuilder](http://slack.k8s.io/#kubebuilder)
* Google Group: [kubebuilder@googlegroups.com](https://groups.google.com/forum/#!forum/kubebuilder)
