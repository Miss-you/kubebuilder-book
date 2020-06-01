# kubebuilder-book

kubebuilder-book 中文翻译，方便国人开发者参考以及使用

kubebuilder 项目链接: https://github.com/kubernetes-sigs/kubebuilder

## Kubebuilder 基础

### 控制器模式

TODO

### 声明式 API

TODO

## kubebuilder 是什么？

Kubebuilder 是一个基于 CRDs 来构建 Kubernetes API 的 SDK 框架，用户可以使用 Kubebuilder 从零开始快速开发和构建 API、Controller 和 Admission Webhook 。其主要功能有：

- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
- 提供代码库封装底层的 K8s go-client；

### 为什么要有 kubebuilder？

目前扩展 Kubernetes 的 API 的方式有创建 CRD、使用 Operator SDK 等方式，都需要写很多的样本文件（boilerplate），使用起来十分麻烦。为了能够更方便构建 Kubernetes API 和工具，就需要一款能够事半功倍的工具，与其他 Kubernetes API 扩展方案相比，kubebuilder 更加简单易用，并获得了社区的广泛支持。

### 工作流程

- 创建一个新的工程目录
- 创建一个或多个资源 API CRD 然后将字段添加到资源
- 在控制器中实现协调循环（reconcile loop），watch 额外的资源
- 在集群中运行测试（自动安装 CRD 并自动启动控制器）
- 更新引导集成测试测试新字段和业务逻辑
- 使用用户提供的 Dockerfile 构建和发布容器

### 设计哲学

- 能使用 go 接口和库，就不使用代码生成
- 能使用代码生成，就不用使用多于一次的存根初始化
- 能使用一次存根，就不 fork 和修改 boilerplate
- 绝不 fork 和修改 boilerplate

## 核心概念

### Kinds & Resources

### API Group & Versions（GV）

### GVKs & GVRs

### Scheme

### Manager

### Cache

### Controller

### Clients

### Index

### Finalizer

### OwnerReference

## kubebuilder 好文推荐

深入解析 Kubebuilder：让编写 CRD 变得更简单: https://www.cnblogs.com/alisystemsoftware/p/11580202.html

## 作者介绍：

yousa，任职于腾讯云，Apache APISIX PMC

### 联系方式：

微信：sytclmissyou
邮箱：yousa@apache.com