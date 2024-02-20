---
title: Composition 资源
weight: 50
description: "复合资源，即一个 XR 或多个 XR，代表相关云资源的集合"
---

复合资源将一组托管资源表示为一个 Kubernetes 对象。 当用户访问 CompositeResourceDefinition 中定义的自定义 API 时，crossplane 就会创建复合资源。

{{<hint "tip" >}}复合资源是托管资源的_组合_，_Composition_定义了如何将托管资源_组合_在一起。{{< /hint >}}

{{<expand "Confused about Compositions, XRDs, XRs and Claims?" >}}crossplane 有四个核心组件，用户通常会把它们混为一谈: 

* [Composition]({{<ref "./compositions">}}) - 用于定义如何创建资源的模板。
* [Composition Resource Definition]({{<ref "./composite-resource-definitions">}}) (`XRD`) - 一种自定义 API 规范。
* Composite Resource (`XR`) - 本页面。通过使用 Composite Resource Definition 中定义的自定义 API 创建。XRs 使用 Composition 模板来创建新的托管资源。
* [声明]({{<ref "./claims" >}}) (`XRC`) - 类似于 Composition Resource，但具有名称空间范围。

{{</expand >}}

## 创建 Composition 资源

创建复合资源需要 [Composition]({{<ref "./compositions">}}) 和 [CompositeResourceDefinition]({{<ref "./composite-resource-definitions">}}Composition 定义了要创建的资源集。 XRD 定义了用户为请求资源集而调用的自定义 API。

Crossplane组件关系图](/media/composition-how-it-works.svg)

XRD 定义了用于创建复合资源的 API。 例如，这个 {{<hover label="xrd1" line="2">}}复合资源定义{{</hover>}}创建了一个自定义 API 端点{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}.

```yaml {label="xrd1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: xMyDatabase
    plural: xmydatabases
  # Removed for brevity
```

当用户调用自定义 API 时、{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}时，Crossplane 会根据 Composition 的{{<hover label="typeref" line="6">}}的{{</hover>}}

```yaml {label="typeref",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: my-composition
spec:
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: xMyDatabase
  # Removed for brevity
```

Composition {{<hover label="typeref" line="6">}}复合类型{{</hover>}}与 XRD {{<hover label="xrd1" line="6">}}组{{</hover>}}和{{<hover label="xrd1" line="9">}}类{{</hover>}}.

crossplane 会创建匹配的 Composition 中定义的资源，并将它们表示为单一的 "复合 "资源。

```shell{copy-lines="1"}
kubectl get composite
NAME SYNCED READY COMPOSITION AGE
my-composite-resource True True my-composition 4s
```

### 命名外部资源

默认情况下，由复合资源创建的托管资源具有复合资源的名称，后跟一个随机后缀。

<!-- vale Google.FirstPerson = NO -->

<!-- vale Crossplane.Spelling = NO -->

例如，名为 "my-composite-resource "的 Composition 资源会创建名为 "my-composite-resource-fqvkw "的外部资源。

<!-- vale Google.FirstPerson = YES -->

<!-- vale Crossplane.Spelling = YES  -->

资源名称可以通过应用{{<hover label="annotation" line="5">}}Annotations{{</hover>}}来确定资源名称。

```yaml {label="annotation",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
  annotations: 
    crossplane.io/external-name: my-custom-name
# Removed for brevity
```

在 Composition 内部，被引用一个{{<hover label="comp" line="10">}}补丁{{</hover>}}将外部名称应用到资源中。

字段路径 {{<hover label="comp" line="11">}}字段路径{{</hover>}}补丁复制了{{<hover label="comp" line="11">}}metadata.Annotations{{</hover>}}字段复制到{{<hover label="comp" line="12">}}metadata.annotations{{</hover>}}字段复制到托管资源内。

{{<hint "note" >}}如果受管资源具有 "crossplane.io/external-name "注解，则 crossplane 会使用该注解值来命名外部资源。{{</hint >}}

```yaml {label="comp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: my-composition
spec:
  resources:
    - name: database
      base:
        # Removed for brevity
      patches:
      - fromFieldPath: metadata.annotations
        toFieldPath: metadata.annotations
```

有关修补资源的更多信息，请参阅 [Patch and Transform]({{<ref "./patch-and-transform">}}) 文档。

### Composition选择器

为复合资源选择一个特定的 Composition，以便与{{<hover label="compref" line="6">}}compositionRef{{</hover>}}

{{<hint "important">}}所选的 Composition 必须允许复合资源使用`compositeTypeRef`字段。 阅读[启用复合资源]({{<ref "./compositions#enabling-composite-resources">}})部分。{{< /hint >}}

```yaml {label="compref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionRef:
    name: my-other-composition
  # Removed for brevity
```

合成资源还可以通过一个{{<hover label="complabel" line="6">}}组成选择器{{</hover>}}.

在 {{<hover label="complabel" line="7">}}matchLabels{{</hover>}}部分提供一个或多个要匹配的 Composition 标签。

```yaml {label="complabel",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionSelector:
    matchLabels:
      environment: production
  # Removed for brevity
```

#### Composition revisions 政策

crossplane 会以 [Composition revisions] 的形式跟踪对 Composition 的修改({{<ref "/knowledge-base/guides/composition-revisions">}}) .

复合资源可以被引用一个 {{<hover label="comprev" line="6">}}compositionUpdatePolicy{{</hover>}}来手动或自动引用较新的 Composition 修订版本。

默认的{{<hover label="comprev" line="6">}}compositionUpdatePolicy{{</hover>}}复合资源会自动被引用最新的 Composition 修订版本。

将政策改为{{<hover label="comprev" line="6">}}手动{{</hover>}}以防止 Composition 资源自动升级。

```yaml {label="comprev",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionUpdatePolicy: Manual
  # Removed for brevity
```

#### Composition revisions 选择器 

crossplane 会把对 Composition 的修改记录为[Composition revisions]({{<ref "/knowledge-base/guides/composition-revisions">}}复合资源可选择特定的 Composition 修订版本。

被引用 {{<hover label="comprevref" line="6">}}compositionRevisionRef{{</hover>}}来按名称选择特定的 Composition Revisions。

例如，要选择特定的 Composition 修订版，请使用所需的 Composition 修订版名称。

```yaml {label="comprevref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionUpdatePolicy: Manual
  compositionRevisionRef:
    name: my-composition-b5aa1eb
  # Removed for brevity
```

{{<hint "note" >}}从中查找 Composition 修订版名称{{<hover label="getcomprev" line="1">}}kubectl get compositionrevision{{</hover>}}

```shell {label="getcomprev",copy-lines="1"}
kubectl get compositionrevision
NAME REVISION XR-KIND XR-APIVERSION AGE
my-composition-5c976ad 1 xmydatabases example.org/v1alpha1 65m
my-composition-b5aa1eb 2 xmydatabases example.org/v1alpha1 64m
```

{{< /hint >}}

合成资源还可以通过一个{{<hover label="comprevsel" line="6">}}组成修订版选择器{{</hover>}}.

在 {{<hover label="comprevsel" line="7">}}matchLabels{{</hover>}}部分提供一个或多个要匹配的 Composition 修订标签。

```yaml {label="comprevsel",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionRevisionSelector:
    matchLabels:
      channel: dev
  # Removed for brevity
```

#### 管理连接secret

当复合资源创建资源时，Crossplane 会提供任何 [connection secrets]({{<ref "./managed-resources#writeconnectionsecrettoref">}}) 提供给 Composition 资源。

{{<hint "important" >}}

资源只能访问 XRD 允许的连接secret。 默认情况下，XRD 允许访问由托管资源生成的所有连接secret。 阅读更多关于[管理连接secret]({{<ref "./composite-resource-definitions#manage-connection-secrets">}}) 的更多信息，请参阅 XRD 文档。{{< /hint >}}

被引用{{<hover label="writesecret" line="6">}}writeConnectionSecretToRef{{</hover>}}来指定 Composition 资源向何处写入连接secret。

例如，该 Composition 资源会将连接 secrets 保存在 Kubernetes secret 对象中，该对象名为{{<hover label="writesecret" line="7">}}我的secret{{</hover>}}的 Kubernetes secret 对象中。{{<hover label="writesecret" line="8">}}crossplane-system{{</hover>}}.

```yaml {label="writesecret",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  writeConnectionSecretToRef:
    name: my-secret
    namespace: crossplane-system
  # Removed for brevity
```

Composition 资源可将连接secret写入[外部secret存储]({{<ref "/knowledge-base/integrations/vault-as-secret-store">}})，如 HashiCorp Vault。

{{<hint "important" >}}外部secret存储是 alpha 功能，默认情况下不启用 alpha 功能。{{< /hint >}}

被引用 {{<hover label="publishsecret"
line="6">}}字段将连接secret保存到外部secret存储中。{{</hover>}}字段将连接secret存储到外部secret存储区。

```yaml {label="publishsecret",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  publishConnectionDetailsTo:
    name: my-external-secret-store
  # Removed for brevity
```

请阅读[外部secret存储]({{<ref "/knowledge-base/integrations/vault-as-secret-store">}}) 文档，了解更多关于使用外部secret存储的信息。

有关连接secret的更多信息，请阅读[连接secret知识库文章]({{<ref "/knowledge-base/guides/connection-details">}}).

### 暂停 Composition 资源

<!-- vale Google.WordList = NO -->

crossplane 支持暂停复合资源。 暂停的复合资源不会检查或更改其外部资源。

<!-- vale Google.WordList = YES -->

要暂停 Composition 资源，请应用{{<hover label="pause" line="4">}}注解。{{</hover>}}注释。

```yaml {label="pause",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
  annotations:
    crossplane.io/paused: "true"
spec:
  # Removed for brevity
```

## 验证 Composition 资源

被引用{{<hover label="getcomposite" line="1">}}kubectl get composite{{</hover>}}查看所有已创建的 Composition 资源。

```shell{copy-lines="1",label="getcomposite"}
kubectl get composite
NAME SYNCED READY COMPOSITION AGE
my-composite-resource True True my-composition 4s
```

对特定自定义 API 端点使用 `kubectl get` 仅可查看这些资源。

```shell {copy-lines="1"}
kubectl get xMyDatabase.example.org
NAME SYNCED READY COMPOSITION AGE
my-composite-resource True True my-composition 12m
```

被引用{{<hover label="desccomposite" line="1">}}kubectl describe composite{{</hover>}}查看链接的{{<hover label="desccomposite" line="16">}}Composition Ref{{</hover>}}中创建的唯一托管资源。{{<hover label="desccomposite" line="22">}}资源引用{{</hover>}}.

```yaml {copy-lines="1",label="desccomposite"}
kubectl describe composite my-composite-resource
Name:         my-composite-resource
API Version:  example.org/v1alpha1
Kind:         xMyDatabase
Spec:
  Composition Ref:
    Name:  my-composition
  Composition Revision Ref:
    Name:                     my-composition-cf2d3a7
  Composition Update Policy:  Automatic
  Resource Refs:
    API Version:  s3.aws.upbound.io/v1beta1
    Kind:         Bucket
    Name:         my-composite-resource-fmrks
    API Version:  dynamodb.aws.upbound.io/v1beta1
    Kind:         Table
    Name:         my-composite-resource-wnr9t
# Removed for brevity
```

#### 综合资源条件

Composition 资源的条件与其 managed 资源的条件相匹配。

详情请阅读托管资源文档中的[条件部分]({{<ref "./managed-resources#conditions">}}) 了解详情。

## Composition 资源标签

Crossplane 为复合资源添加标签，以显示它们与其他 Crossplane 组件的关系。

#### 合成标签

crossplane 添加了{{<hover label="complabel" line="4">}}crossplane.io/composite{{</hover>}}标签。 该标签与合成资源的名称相匹配。 Crossplane 会将合成标签应用到由合成资源创建的任何托管资源，从而在托管资源和拥有合成资源的资源之间创建一个引用。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/composite=my-claimed-database-x9rx9
```

### claim名称标签

crossplane 添加了{{<hover label="claimname" line="4">}}crossplane.io/claim-name{{</hover>}}标签，该标签表示链接到此复合资源的 claims 的名称。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/claim-name=my-claimed-database
```

直接创建的 Composition 资源不被引用 claim，因此没有{{<hover label="claimname" line="4">}}crossplane.io/claim-name{{</hover>}}标签。

### claim名称空间标签

crossplane 添加了{{<hover label="claimname" line="4">}}crossplane.io/claim-namespace{{</hover>}}标签，以指示与该复合资源链接的 claims 的 namespace。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/claim-namespace=default
```

直接创建的 Composition 资源不被引用 claim，因此没有{{<hover label="claimname" line="4">}}crossplane.io/claim-namespace{{</hover>}}标签。
