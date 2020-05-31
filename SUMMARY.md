# Summary

[介绍](./introduction.md)

[快速入门](./quick-start.md)

---

- [教程:构建 CronJob ](cronjob-tutorial/cronjob-tutorial.md)

  - [基础项目组件介绍](./cronjob-tutorial/basic-project.md)
  - [每段旅程需要一个起点，每个程序需要一个入口函数](./cronjob-tutorial/empty-main.md)
  - [GVK 介绍](./cronjob-tutorial/gvks.md)
  - [添加一个 API](./cronjob-tutorial/new-api.md)
  - [设计一个 API](./cronjob-tutorial/api-design.md)

      - [浅谈: What's the rest of this stuff?](./cronjob-tutorial/other-api-files.md)

  - [控制器简介](./cronjob-tutorial/controller-overview.md)
  - [实现控制器](./cronjob-tutorial/controller-implementation.md)

    - [You said something about main?](./cronjob-tutorial/main-revisited.md)

  - [实现默认/校验 webhooks](./cronjob-tutorial/webhook-implementation.md)
  - [运行和部署控制器](./cronjob-tutorial/running.md)

    - [部署证书管理器](./cronjob-tutorial/cert-manager.md)
    - [部署 webhooks ](./cronjob-tutorial/running-webhook.md)

  - [后记](./cronjob-tutorial/epilogue.md)

- [教程:多版本 API](./multiversion-tutorial/tutorial.md)

  - [api 变化之处](./multiversion-tutorial/api-changes.md)
  - [Hubs, spokes, and other wheel metaphors](./multiversion-tutorial/conversion-concepts.md)
  - [转换](./multiversion-tutorial/conversion.md)

      - [设置 webhooks](./multiversion-tutorial/webhooks.md)

  - [部署和测试](./multiversion-tutorial/deployment.md)

---

- [迁移](./migrations.md)

  - [Kubebuilder v1/v2版本对比](./migration/v1vsv2.md)

      - [迁移教程](./migration/guide.md)

  - [单组转成多组](./migration/multi-group.md)

---

- [参考资料](./reference/reference.md)

  - [生成 CRDs](./reference/generating-crd.md)
  - [使用 Finalizers](./reference/using-finalizers.md)
  - [Kind 集群](reference/kind.md)
  - [什么是 webhook?](reference/webhook-overview.md)
    - [Admission webhook](reference/admission-webhook.md)
    - [Webhooks 核心类型](reference/webhook-for-core-types.md)
  - [用于配置/代码生成的标记](./reference/markers.md)

      - [CRD 生成](./reference/markers/crd.md)
      - [CRD 校验](./reference/markers/crd-validation.md)
      - [CRD 处理](./reference/markers/crd-processing.md)
      - [Webhook](./reference/markers/webhook.md)
      - [对象/深拷贝](./reference/markers/object.md)
      - [RBAC](./reference/markers/rbac.md)

  - [控制器生成工具 CLI](./reference/controller-gen.md)
  - [completion](./reference/completion.md)
  - [工具](./reference/artifacts.md)
  - [编写控制器测试用例](./reference/writing-tests.md)

    - [在集成测试中使用envtest](./reference/testing/envtest.md)

  - [Metrics](./reference/metrics.md)

---

[附录: TODO 列表](./TODO.md)
