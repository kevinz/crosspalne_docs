---

title: 功能生命周期
toc: true
weight: 309
缩进: true

---

# 功能生命周期

crossplane 采用与[上游 Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages)类似的功能生命周期。所有重要的新功能都必须在 alpha 版中添加。预计 alpha 版功能最终会升级到 beta 版，然后再升级到通用版（GA）。在 alpha 版或 beta 版徘徊的功能可能会被淘汰。

## alpha 功能

默认情况下，alpha 功能是关闭的，必须通过功能 flag（例如 `--enable-composition-revisions`）启用。 与 alpha 功能相关的 API 类型使用 `vNalphaN` 样式的 API 版本，例如 `v1alpha`。 **alpha 功能可能会被移除或进行破坏性修改，恕不另行通知**，通常不被认为可用于生产。

在某些情况下，alpha 功能需要将字段添加到现有的 beta 或 GA API 类型中。 在这些情况下，字段必须明确标记（即在其 OpenAPI 架构中）为 alpha，并受 alpha API 约束（或无约束）。

所有 alpha 功能都应该有一个问题跟踪，以追踪它们是否能升级到 beta 版。

## 测试版功能

测试版功能默认开启，但也可通过功能 flag 禁用。 与测试版功能相关的 API 类型使用 `vNbetaN` 样式的 API 版本，如 `v1beta1`。 测试版功能被认为是经过良好测试的，在未标记为至少两个发布版本的过时功能之前，不会被完全删除。

在随后的测试版或稳定版中，对象的模式和/或语义可能会以不兼容的方式发生变化。 当发生这种情况时，我们将提供迁移到下一版本的说明。 这可能需要删除、编辑和重新创建 API 对象。 编辑过程可能需要一些思考。 这可能需要依赖该功能的应用程序停机。

在某些情况下，测试版功能需要将字段添加到现有的 GA API 类型中。 在这些情况下，字段必须明确标记（即在其 OpenAPI 架构中）为测试版，并受测试版 API 约束（或无约束）。

所有测试版功能都应该有一个问题跟踪，以追踪它们是否能升级到 GA。

## GA 功能

GA 功能总是启用的，不能禁用。 与 GA 功能相关的 API 类型使用 `vN` 风格的 API 版本，如 `v1`。 GA 功能被广泛引用并经过全面测试。 它们保证了 API 的稳定性，只允许向后兼容的更改。