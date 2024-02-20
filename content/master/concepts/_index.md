---

title: 概念
weight: 100
说明: 了解 Crossplane 的核心组件

---

Crossplane 引入了多个构建模块，使您能够使用 Kubernetes API 对基础架构进行调配、编译和消费。 这些单独的概念相互配合，在组织中的不同角色之间实现了强大的关注分离，这意味着团队中的每个成员都能在适当的抽象层次上与 Crossplane 进行交互。

## Packages 

[Packages]允许将 Crossplane 扩展自新功能。 这通常表现为捆绑一组 Kubernetes [CRDs] 和 [控制器]，用于代表和管理外部基础设施（即 Provider），然后将它们安装到运行 Crossplane 的集群中。 Crossplane 会确保新的 CRDs 不会与现有的 CRDs 冲突，并管理新包的 RBAC 和安全性。 严格来说，Packages 并不要求必须是 Provider，但这是目前最常见的包用例。

## Providers

Providers 是使 Crossplane 能够在外部服务上配置基础架构的软件包。 它们带来了一对一映射到外部基础架构资源的 CRD（即托管资源），以及管理这些资源生命周期的控制器。 你可以在[providers 文档]中阅读有关 Providers 的更多信息，包括如何安装和配置它们。

## 受托管的资源

受托管资源是 Kubernetes 的自定义资源，代表基础设施原语。 API 版本为 `v1beta1` 或更高的托管资源支持云提供商为给定资源所做的每个字段。 您可以在 [Upbound Marketplace] 上找到每个提供商的托管资源及其 API 规范，并在 [托管资源文档] 中了解更多信息。

## Composition Resources

复合资源 (XR) 是一种特殊的自定义资源，由 "复合资源定义"（CompositeResourceDefinition）定义。 它将一个或多个托管资源composition一个更高级别的基础架构单元。 复合资源面向基础架构操作员，但可选择提供面向应用程序开发人员的复合资源声明，作为复合资源的代理。 您可以在[composition文档]中了解有关所有这些概念的更多信息。

<!-- Named Links -->

[Packages]:  {{<ref "packages" >}}[CRDs]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/ [controllers]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers [providers documentation]:  {{<ref "providers" >}}[Upbound Marketplace]: https://marketplace.upbound.io [managed resources documentation]:  {{<ref "managed-resources" >}}[Composition文档]:  {{<ref "./compositions" >}}
