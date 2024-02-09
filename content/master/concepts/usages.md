---

title: Usage
weight: 95
状态:  alpha
alphaVersion: "1.14"
description: "Usage（使用）定义了托管资源或 Composition 的使用关系"

---

Usage "是一种 crossplane 资源，它定义了受管资源或 Composition 资源的使用关系。 Usage 的两个主要用例如下: 

1.保护资源不被意外删除。
2.通过确保资源不会在其从属资源被删除之前被删除，实现删除排序。

第一个用例请参阅[删除保护的 Usage]（#usage-for-deletion-protection）一节，第二个用例请参阅[删除排序的 Usage]（#usage-for-deletion-ordering）一节。

## 启用 Usage

Usage 是一项 alpha 功能，默认情况下不会启用。

通过[更改 crossplane pod 设置]（）启用 "Usage "支持。{{<ref "./pods#change-pod-settings">}}) 并启用{{<hover label="deployment" line="12">}}--启用 Usage{{</hover>}}参数。

```yaml {label="deployment",copy-lines="12"}
$ kubectl edit deployment crossplane --namespace crossplane-system
apiVersion: apps/v1
kind: Deployment
spec:
# Removed for brevity
  template:
    spec:
      containers:
      - args:
        - core
        - start
        - --enable-usages
```

{{<hint "tip" >}}

crossplane 安装指南]({{<ref "../software/install#feature-flags">}}) 介绍了如何启用功能标志，如{{<hover label="deployment" line="12">}}--启用 Usage{{</hover>}}等功能标志。{{< /hint >}}

<!-- vale Google.Headings = NO -->

## 创建 Usage

<!-- vale Google.Headings = YES -->

A {{<hover label="protect" line="2">}}Usage{{</hover>}}{{<hover label="protect" line="5">}}规格{{</hover>}}有一个强制性的{{<hover label="protect" line="6">}}的{{</hover>}}字段，用于定义被引用或被保护的资源。{{<hover label="protect" line="11">}}字段{{</hover>}}字段定义了保护的原因，而 {{<hover label="order" line="11">}}字段{{</hover>}}字段

<!-- vale write-good.Passive = NO -->

定义了使用资源。 这两个字段都是可选的，但至少必须提供其中一个。

<!-- vale write-good.Passive = YES -->

{{<hint "important" >}}

<!-- vale write-good.Passive = NO -->

Usage 关系可以在 "受管资源 "和 "Composition "之间定义。

<!-- vale write-good.TooWordy = NO -->

但是，除非使用 `compositeDeletePolicy` `Foreground`，否则作为使用资源（`spec.by`）的 `Composition` 将不起作用，因为它不会在使用默认删除策略 `Background` 删除自己的资源之前阻止删除其子资源。

<!-- vale write-good.TooWordy = YES -->

<!-- vale write-good.Passive = YES -->

{{< /hint >}}

#### 删除保护的 Usage

以下示例可防止删除{{<hover label="protect" line="10">}}我的数据库{{</hover>}}资源的删除。{{<hover label="protect" line="11">}}原因{{</hover>}}定义的删除请求。

```yaml {label="protect"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: protect-production-database
spec:
  of:
    apiVersion: rds.aws.upbound.io/v1beta1
    kind: Instance
    resourceRef:
      name: my-database
  reason: "Production Database - should never be deleted!"
```

#### 删除排序的 Usage

以下示例可防止删除{{<hover label="order" line="10">}}我的集群{{</hover>}}资源之前拒绝任何删除请求，从而防止删除{{<hover label="order" line="15">}}我的普罗米修斯图表{{</hover>}}资源。

```yaml {label="order"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceRef:
      name: my-cluster
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceRef:
      name: my-prometheus-chart
```

### 使用带有 Usage 的选择器

Usage 可以被引用为 {{<hover label="selectors" line="9">}}选择器{{</hover>}}来定义被引用或使用的资源。 这样就可以使用 {{<hover label="selectors" line="12">}}标签{{</hover>}}或{{<hover label="selectors" line="10">}}匹配控制器引用{{</hover>}}来定义资源，而不是提供资源名称。

```yaml {label="selectors"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceSelector:
      matchControllerRef: false # default, and could be omitted
      matchLabels:
        foo: bar
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceSelector:
       matchLabels:
          baz: qux
```

Usage 控制器解析选择器后，会将资源名称持久化到{{<hover label="selectors-resolved" line="10">}}资源名称。{{</hover>}}下面的示例显示了解析选择器后的 `Usage` 资源。

{{<hint "important" >}}

<!-- vale write-good.Passive = NO -->

选择器只解析一次，如果有多个匹配项，则从匹配资源列表中随机选择一个资源。

<!-- vale write-good.Passive = YES -->

{{< /hint >}}

```yaml {label="selectors-resolved"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceRef:
       name: my-cluster
    resourceSelector:
      matchLabels:
        foo: bar
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceRef:
       name: my-cluster
    resourceSelector:
       matchLabels:
          baz: qux
```

## Usage in a Composition

Usage 的典型用例是定义 Composition 中资源之间的删除顺序。 Usage 支持选择器中的[匹配控制器引用]({{<ref "./compositions#match-a-controller-reference" >}})，以确保匹配资源与[跨资源引用]({{<ref "./compositions#cross-resource-references" >}}).

下面的示例显示了一个 Composition，该 Composition 在 "集群 "和 "发布 "资源之间定义了删除顺序。 Usage "阻止删除 "集群 "资源，直到 "发布 "资源被成功删除。

```yaml {label="composition"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: cluster
      base:
        apiVersion: container.gcp.upbound.io/v1beta1
        kind: Cluster
        # Removed for brevity
    - name: release
      base:
        apiVersion: helm.crossplane.io/v1beta1
        kind: Release
        # Removed for brevity
    - name: release-uses-cluster
      base:
        apiVersion: apiextensions.crossplane.io/v1alpha1
        kind: Usage
        spec:
          of:
            apiVersion: container.gcp.upbound.io/v1beta1
            kind: Cluster
            resourceSelector:
              matchControllerRef: true
          by:
            apiVersion: helm.crossplane.io/v1beta1
            kind: Release
            resourceSelector:
              matchControllerRef: true
```

{{<hint "tip" >}}

<!-- vale write-good.Passive = NO -->

当 Composition 中有多个相同类型的资源时，"Usage"（使用情况）选项会显示"......"。{{<hover label="composition" line="18">}}Usage{{</hover>}}资源必须唯一标识正在使用或正在使用的资源。 这可以通过使用额外的标签和结合{{<hover label="composition" line="24">}}matchControllerRef{{</hover>}}另一种方法是借助 `ToCompositeFieldPath` 和 `FromCompositeFieldPath` 或 `ToEnvironmentFieldPath` 和 `FromEnvironmentFieldPath` 类型的补丁，直接对 `resourceRef.name` 进行修补。

<!-- vale write-good.Passive = YES -->

{{< /hint >}}
