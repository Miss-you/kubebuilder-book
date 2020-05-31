# kubebuilder-book

kubebuilder-book 中文翻译，方便国人开发者参考以及使用

kubebuilder 项目链接: https://github.com/kubernetes-sigs/kubebuilder

## Kubebuilder 基础

### 控制器模式

TODO

### 声明式 API

TODO

## kubebuilder 是什么？

Kubebuilder 是一个使用 CRDs 构建 Kubernetes API 的 SDK，主要功能有：

- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
- 提供代码库封装底层的 K8s go-client；

方便用户从零开始开发 CRDs、Controllers 和 Admission Webhooks 来扩展 Kubernetes

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