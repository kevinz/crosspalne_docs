---

title: "概述"
weight: -1
cascade: 
    版本: "master"

---

{{< img src="/media/banner.png" alt="Crossplane Popsicle Truck" size="large" >}}

<br />

Crossplane 是 Kubernetes 的开源扩展，可将 Kubernetes 集群转化为**通用控制平面。

Crossplane 可让您通过标准的 Kubernetes API 来管理任何地方的任何东西。 Crossplane 甚至可以让您直接从 Kubernetes [订购披萨](https://blog.crossplane.io/providers-101-ordering-pizza-with-kubernetes-and-crossplane/)。只要它有 API，Crossplane 就能连接到它。

借助 Crossplane，平台团队可以利用 Kubernetes 策略、namespace、基于角色的访问控制等强大功能创建新的抽象和自定义 API。 Crossplane 将所有非 Kubernetes 资源集中在一个平台上。

由平台团队创建的自定义 API 允许跨资源或云执行安全性和合规性，而不会向开发人员暴露任何复杂性。 单个 API 调用就可以在多个云中创建多个资源，并将 Kubernetes 用作一切的控制平面。

{{< hint "tip" >}}
**什么是控制平面？

<!-- vale Google.WordList = NO -->

控制平面创建并管理资源的生命周期，不断检查预期资源是否存在，当预期状态与现实不符时进行报告，并采取行动纠正错误。

crossplane 扩展了 Kubernetes 控制平面，使其成为**通用的控制平面**，可对任何地方的任何资源进行检查、报告和操作。

<!-- vale Google.WordList = YES -->

{{< /hint >}}

# 开始

* 在 Kubernetes 集群中安装 [Install Crossplane]({{<ref "software/install">}})安装在你的 Kubernetes 集群中
* 了解有关 Crossplane 如何工作的更多信息，请参阅

[crossplane介绍]({{<ref "getting-started/introduction" >}})

* 加入 [Crossplane Slack](https://slack.crossplane.io/) 并启动

与 7000 多名运营商进行对话。

crossplane 是一个 [Cloud Native Compute Foundation](https://www.cncf.io/) 项目。