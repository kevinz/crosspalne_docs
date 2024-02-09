---

title: 环境配置
weight: 75
state: alpha
alphaVersion: "1.11"
description: "环境配置或 EnvironmentConfigs 是一个内存数据存储，在修补 Composition 时被引用"

---

<!--
TODO: Add Policies
-->

crossplane EnvironmentConfig 是一种类似于集群范围的 [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) 的资源，由 Composition 使用。Composition 可以使用环境来存储单个资源的信息或引用 [patches]({{<ref "patch-and-transform">}}).

crossplane 支持多个 EnvironmentConfigs，每个都是唯一的数据存储。

Crossplane 创建复合资源时，会合并相关 Composition 中引用的所有 EnvironmentConfigs，并为该复合资源创建一个唯一的内存环境。

Composition 资源可以在其独特的内存环境中读写数据。

{{<hint "important" >}}每个 Composition 资源的内存环境都是唯一的。 一个 Composite 资源无法读取另一个 Composite 资源环境中的数据。{{< /hint >}}

## 启用环境配置

EnvironmentConfigs 是一项 alpha 功能，默认情况下不会启用。

通过[更改 crossplane pod 设置]({{<ref "./pods#change-pod-settings">}}) 并启用{{<hover label="deployment" line="12">}}--参数{{</hover>}}参数。

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
        - --enable-environment-configs
```

{{<hint "tip" >}}

crossplane 安装指南]({{<ref "../software/install#feature-flags">}}) 介绍了如何启用功能标志，如{{<hover label="deployment" line="12">}}--启用环境配置{{</hover>}}等功能标志。{{< /hint >}}

<!-- vale Google.Headings = NO -->

## 创建环境配置

<!-- vale Google.Headings = YES -->

一个 {{<hover label="env1" line="2">}}环境配置{{</hover>}}有一个对象字段、{{<hover label="env1" line="5">}}数据{{</hover>}}.

环境配置支持{{<hover label="env1" line="5">}}数据{{</hover>}}字段内的任何数据。

下面是一个示例{{<hover label="env1" line="2">}}环境配置{{</hover>}}.

```yaml {label="env1"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: example-environment
data:
  locations:
    us: us-east-2
    eu: eu-north-1
  key1: value1
  key2: value2
  key3:
    - item1
    - item2
```

<!-- vale Google.Headings = NO -->

## 选择环境配置

<!-- vale Google.Headings = YES -->

选择要在 Composition 内部引用的 EnvironmentConfigs{{<hover label="comp" line="6">}}环境{{</hover>}}字段。

环境配置 {{<hover label="comp" line="7">}}environmentConfigs{{</hover>}}字段是该 Composition 可以引用的环境列表。

通过以下方式选择环境{{<hover label="comp" line="8">}}参考{{</hover>}}或{{<hover label="comp" line="11">}}选择器{{</hover>}}.

A{{<hover label="comp" line="8">}}参考{{</hover>}}通过{{<hover label="comp" line="10">}}名称来选择环境。{{</hover>}}的{{<hover label="comp" line="11">}}选择器{{</hover>}}根据{{<hover label="comp" line="13">}}标签{{</hover>}}应用于环境。

```yaml {label="comp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Reference
      ref:
        name: example-environment
    - type: Selector
      selector:
        matchLabels:
      # Removed for brevity
```

如果一个 Composition 被引用多个{{<hover label="comp" line="7">}}环境配置{{</hover>}}crossplane 会按照列出的顺序将它们合并在一起。

{{<hint "note" >}}如果多个{{<hover label="comp" line="7">}}environmentConfigs{{</hover>}}如果多个 environmentConfigs 使用相同的键，则 Composition 会引用最后列出的环境的值。{{</hint >}}

#### 按名称选择

通过名称选择环境{{<hover label="byName" line="8">}}类型:  参考{{</hover>}}.

定义{{<hover label="byName" line="9">}}对象和{{</hover>}}对象和{{<hover label="byName" line="10">}}名称{{</hover>}}与环境的确切名称相匹配。

例如，选择{{<hover label="byName" line="7">}}环境配置{{</hover>}}名为{{<hover label="byName" line="10">}}example-environment{{</hover>}}

```yaml {label="byName",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Reference
      ref:
        name: example-environment
```

### 按标签选择

通过带有{{<hover label="byLabel" line="8">}}类型: 选择器{{</hover>}}.

定义 {{<hover label="byLabel" line="9">}}选择器{{</hover>}}对象。

匹配标签{{<hover label="byLabel" line="10">}}matchLabels{{</hover>}}对象包含一个要匹配的标签列表。

选择标签需要同时匹配标签{{<hover label="byLabel" line="11">}}键{{</hover>}}和 key 的值。

匹配标签值时，请提供一个精确的值，并用{{<hover label="byLabel" line="12">}}类型: Values{{</hover>}}并在{{<hover label="byLabel" line="13">}}Values{{</hover>}}字段中提供要匹配的值。

crossplane 还可以根据 Composition 资源中的输入匹配标签值。 使用{{<hover label="byLabel" line="15">}}类型: FromCompositeFieldPath{{</hover>}}并在{{<hover label="byLabel" line="16">}}字段中提供要匹配的字段。{{</hover>}}字段中提供要匹配的字段。

```yaml {label="byLabel",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector: 
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
  resources:
  # Removed for brevity
```

#### 管理选择器结果

按标签选择环境可能会返回多个环境。 Composition 会按环境名称对所有结果进行排序，并只引用排序列表中的第一个环境。

设置 {{<hover label="selectResults" line="10">}}模式{{</hover>}}为{{<hover label="selectResults" line="10">}}模式: 多重{{</hover>}}以返回所有匹配的环境。 使用{{<hover label="selectResults" line="19">}}模式: 单{{</hover>}}返回单个环境。

{{<hint "note" >}}排序和选择{{<hover label="selectResults" line="10">}}模式{{</hover>}}仅适用于单个{{<hover label="selectResults" line="8">}}类型: 选择器{{</hover>}}.

这不会改变 Composition 合并多个{{<hover label="selectResults" line="7">}}环境配置{{</hover>}}.{{< /hint >}}

```yaml {label="selectResults"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector:
        mode: Multiple
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
    - type: Selector
      selector:
        mode: Single
        matchLabels:
          - key: my-other-label-key
            type: Value
            value: my-other-label-value
          - key: my-other-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
```

被引用时{{<hover label="maxMatch" line="10">}}模式: 多重{{</hover>}}时，使用{{<hover label="maxMatch" line="11">}}maxMatch{{</hover>}}并定义返回环境的最大数量。

使用 `minMatch` 并定义返回环境的最小数量。

Composition 按名称的字母顺序对返回的环境排序。 使用{{<hover label="maxMatch" line="12">}}sortByFieldPath{{</hover>}}并定义要排序的字段。

```yaml {label="maxMatch"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector:
        mode: Multiple
        maxMatch: 4
        sortByFieldPath: metadata.annotations[sort.by/weight]
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
```

选择的环境{{<hover label="maxMatch" line="18">}}匹配标签{{</hover>}}中列出的任何其他环境进行合并。{{<hover label="maxMatch" line="7">}}环境配置{{</hover>}}.

#### 可选的选择器标签

默认情况下，如果一个{{<hover label="byLabelOptional" line="16">}}valueFromFieldPath{{</hover>}}字段不存在于 Composition 资源中时，Crossplane 会出错。

添加{{<hover label="byLabelOptional" line="17">}}fromFieldPathPolicy{{</hover>}}作为 {{<hover label="byLabelOptional" line="17">}}可选{{</hover>}}以忽略不存在的字段。

```yaml {label="byLabelOptional",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
      - type: Selector
        selector:
          matchLabels:
            - key: my-first-label-key
              type: Value
              value: my-first-label-value
            - key: my-second-label-key
              type: FromCompositeFieldPath
              valueFromFieldPath: spec.parameters.deploy
              fromFieldPathPolicy: Optional
  resources:
  # Removed for brevity
```

为可选标签设置默认值，方法是设置默认值{{<hover label="byLabelOptionalDefault" line="15">}}Values{{</hover>}}的{{<hover label="byLabelOptionalDefault" line="14">}}键的默认值{{</hover>}}的默认值，然后定义{{<hover label="byLabelOptionalDefault" line="20">}}可选{{</hover>}}标签。

例如，该 Composition 定义了{{<hover label="byLabelOptionalDefault" line="16">}}Values: my-default-value{{</hover>}}为键 {{<hover label="byLabelOptionalDefault" line="14">}}my-second-label-key 的值。{{</hover>}}如果标签{{<hover label="byLabelOptionalDefault" line="17">}}my-second-label-key{{</hover>}}存在，则 crossplane 会引用该标签的值。

```yaml {label="byLabelOptionalDefault",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
      - type: Selector
        selector:
          matchLabels:
            - key: my-first-label-key
              type: Value
              value: my-label-value
            - key: my-second-label-key
              type: Value
              value: my-default-value
            - key: my-second-label-key
              type: FromCompositeFieldPath
              valueFromFieldPath: spec.parameters.deploy
              fromFieldPathPolicy: Optional
  resources:
  # Removed for brevity
```

{{<hint "warning" >}}crossplane 按顺序应用 Values 值，最后定义的键值总是优先。

在标签之后定义默认值总是会覆盖标签值。{{< /hint >}}

## 使用 EnvironmentConfigs 进行修补

当 crossplane 创建或更新复合资源时，Crossplane 会将所有指定的 EnvironmentConfigs 合并到内存环境中。

复合资源可以在环境配置和复合资源之间或环境配置和复合资源内部定义的单个资源之间读写数据。

{{<hint "tip" >}}在 [补丁和转换]({{<ref "./patch-and-transform">}}) 文档中有关环境配置补丁类型的信息。{{< /hint >}}

<!-- these two sections are duplicated in the compositions doc with different header depths -->

#### 修补一个 Composition 资源

要给 Composition 资源打补丁，请引用{{< hover label="xrpatch" line="7">}}补丁{{</hover>}}内的{{< hover label="xrpatch" line="5">}}环境{{</hover>}}.

被引用{{< hover label="xrpatch" line="5">}}到复合字段路径{{</hover>}}将数据从内存环境复制到 Composition 资源。 使用{{< hover label="xrpatch" line="5">}}从复合字段路径{{</hover>}}将数据从 Composition 资源复制到内存环境。

```yaml {label="xrpatch",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
  # Removed for Brevity
      patches:
      - type: ToCompositeFieldPath
        fromFieldPath: tags
        toFieldPath: metadata.labels[envTag]
      - type: FromCompositeFieldPath
        fromFieldPath: metadata.name
        toFieldPath: newEnvironmentKey
```

单个资源可以使用写入内存环境的任何数据。

#### 对个别资源进行修补

要为单个资源打补丁，请在{{<hover label="envpatch" line="16">}}补丁{{</hover>}}内，使用{{<hover label="envpatch" line="17">}}环境字段路径{{</hover>}}将数据从资源复制到内存环境。 使用 {{<hover label="envpatch" line="20">}}从环境字段路径{{</hover>}}将数据从内存环境复制到资源。

```yaml {label="envpatch",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
  # Removed for Brevity
  resources:
  # Removed for Brevity
    - name: vpc
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        spec:
          forProvider:
            cidrBlock: 172.16.0.0/16
      patches:
        - type: ToEnvironmentFieldPath
          fromFieldPath: status.atProvider.id
          toFieldPath: vpcId
        - type: FromEnvironmentFieldPath
          fromFieldPath: tags
          toFieldPath: spec.forProvider.tags
```

补丁和转换]({{<ref "./patch-and-transform">}}) 文档中有更多关于修补单个资源的信息。

<!-- End duplicated content -->
