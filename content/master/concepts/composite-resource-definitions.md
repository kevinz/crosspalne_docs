---

title: Composition 资源定义
weight: 40
description: "复合资源定义或 XRD 定义自定义 API 模式"

---

Composite resource definitions (`XRD`)定义了自定义 API 的模式。 用户使用由`XRD`定义的 API 模式创建 Composite resources (`XRs`)和 Claims (`XCs`)。

{{< hint "note" >}}

请阅读 [Composition resources]({{<ref "./composite-resources">}}) 页面，了解有关 Composition 资源的更多信息。

请阅读 [Claims]({{<ref "./claims">}}) 页面获取更多有关claim的信息。{{</hint >}}

{{<expand "Confused about Compositions, XRDs, XRs and Claims?" >}}crossplane 有四个核心组件，用户通常会把它们混为一谈: 

* [Composition]({{<ref "./compositions" >}}) - 用于定义如何创建资源的模板。
* Composite Resource Definition (`XRD`) - 本页面。自定义 API 规范。
* [复合资源]({{<ref "./composite-resources">}}) (`XR`) - 通过使用 Composition Resource Definition 中定义的自定义 API 创建。XRs 使用 Composition 模板来创建新的托管资源。
* [claim]({{<ref "./claims" >}}) (`XRC`) - 类似于 Composition Resource，但具有名称空间范围。

{{</expand >}}

Crossplane XRD 类似于[Kubernetes 自定义资源定义](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。XRD 需要的字段较少，并添加了与 Crossplane 相关的选项，如 claims 和连接secret。

## 创建复合资源定义

创建 CompositeResourceDefinition 的步骤包括

* [定义自定义 API 组](#xrd-groups)。
* [定义自定义 API 名称](#xrd-names)。
* [定义自定义 API 模式和版本](#xrd-versions)。

CompositeResourceDefinitions 还支持可选项: 

* [提供claim](#enable-claims)。
* [定义连接secret](#manage-connection-secrets)。
* [设置 Composition 资源默认值](#set-composite-resource-defaults)。

Composition 资源定义（"XRD"）可在 Kubernetes 集群内创建新的 API 端点。

创建新的应用程序接口需要定义一个应用程序接口{{<hover label="xrd1" line="6">}}组{{</hover>}},{{<hover label="xrd1" line="7">}}名称{{</hover>}}和{{<hover label="xrd1" line="10">}}版本{{</hover>}}.

```yaml {label="xrd1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: XMyDatabase
    plural: xmydatabases
  versions:
  - name: v1alpha1
  # Removed for brevity
```

应用 XRD 后，crossplane 会创建一个新的 Kubernetes 自定义资源定义，以匹配所定义的 API。

例如，XRD{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover >}}创建一个自定义资源定义{{<hover label="kubeapi" line="2">}}xmydatabases.example.org{{</hover >}}.

```shell {label="kubeapi",copy-lines="3"}
kubectl api-resources
NAME SHORTNAMES APIVERSION NAMESPACED KIND
xmydatabases.example.org v1alpha1 false xmydatabases
# Removed for brevity
```

{{<hint "warning">}}您不能更改 XRD{{<hover label="xrd1" line="6">}}组{{</hover>}}或{{<hover label="xrd1" line="7">}}名称。{{</hover>}}您必须删除并重新创建 XRD 才能更改{{<hover label="xrd1" line="6">}}组{{</hover>}}或{{<hover label="xrd1" line="7">}}名称{{</hover>}}.{{</hint >}}

### XRD 组

groups 定义了相关 API 端点的集合，"group "可以是任何值，但常见的惯例是映射到完全合格的域名。

<!-- vale write-good.Weasel = NO -->

许多 XRD 可能会被引用相同的 "组 "来创建 API 的逻辑集合。

<!-- vale write-good.Weasel = YES -->

例如，"数据库 "组可能有 "关系 "和 "nosql "两种。

{{<hint "tip" >}}组名称具有集群作用域。 选择不与 Providers 冲突的组名称。 避免在组中使用 Provider 名称。{{< /hint >}}

### XRD 名称

名称 "字段定义了如何引用此特定 XRD。 所需的名称字段有

* `kind` - 调用此 API 时被引用的 `kind` 值。种类是 [UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects)。crossplane 建议 XRD `kinds` 以 `X` 开头，以显示它是自定义的 crossplane API 定义。
* `plural` - API URL 被引用的复数名称。复数名称必须小写。

{{<hint "important" >}}XRD{{<hover label="xrdName" line="4">}}元数据名称{{</hover>}}必须是{{<hover label="xrdName" line="9">}}复数{{</hover>}}name, `.`（点字符）、{{<hover label="xrdName" line="6">}}组{{</hover>}}.

例如 {{<hover label="xrdName"
line="4">}}xmydatabases.example.org{{</hover>}}匹配 {{<hover
label="xrdName" line="9">}}复数{{</hover>}}名称{{<hover label="xrdName" line="9">}}xmydatabases{{</hover>}}, `.`{{<hover label="xrdName" line="6">}}组{{</hover>}}名称、{{<hover label="xrdName" line="6">}}example.org{{</hover>}}.

```yaml {label="xrdName",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: XMyDatabase
    plural: xmydatabases
    # Removed for brevity
```

{{</hint >}}

### XRD 版本

<!-- vale gitlab.SentenceLength = NO -->

XRD `version` 就像[Kubernetes 所引用的 API 版本](https://kubernetes.io/docs/reference/using-api/#api-versioning)。版本显示了 API 的成熟或稳定程度，并在更改、添加或删除 API 中的字段时递增。

<!-- vale gitlab.SentenceLength = YES -->

crossplane 并不要求特定的版本或特定的版本命名约定，但强烈建议遵循[Kubernetes API 版本指南](https://kubernetes.io/docs/reference/using-api/#api-versioning)。

* `v1alpha1` - 随时可能更改的新 API。
* `v1beta1` - 稳定的现有 API。不鼓励进行破坏性更改。
* `v1` - 没有破坏性更改的稳定 API。

#### 定义模式

<!-- vale write-good.Passive = NO -->

<!-- vale write-good.TooWordy = NO -->

模式 "定义了参数的名称、参数的数据类型以及哪些参数是必需的，哪些是可选的。

<!-- vale write-good.Passive = YES -->

<!-- vale write-good.TooWordy = YES -->

{{<hint "note" >}}所有 "模式 "都遵循 Kubernetes 自定义资源定义[OpenAPIv3 结构模式](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema)。{{< /hint >}}

每个{{<hover label="schema" line="11">}}版本{{</hover>}}都有一个独特的{{<hover label="schema" line="12">}}模式{{</hover>}}.

所有 XRD {{<hover label="schema" line="12">}}模式{{</hover>}}根据 {{<hover label="schema" line="13">}}模式进行验证。{{</hover>}}模式是 OpenAPI{{<hover label="schema" line="14">}}对象{{</hover>}}对象，具有{{<hover label="schema" line="15">}}属性{{</hover>}}属性的{{<hover label="schema" line="16">}}模式{{</hover>}}{{<hover label="schema" line="17">}}对象{{</hover>}}.

在 {{<hover label="schema" line="18">}}spec.properties{{</hover>}}是自定义 API 的定义。

在本例中，关键字 {{<hover label="schema" line="19">}}区域{{</hover>}}是一个 {{<hover label="schema" line="20">}}字符串{{</hover>}}.

```yaml {label="schema",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string
    # Removed for brevity
```

使用该 API 的 Composition 资源会引用{{<hover label="xr" line="1">}}组/版本{{</hover>}}和{{<hover label="xr" line="2">}}种类。{{</hover>}}的{{<hover label="xr" line="5">}}规格{{</hover>}}有{{<hover label="xr" line="6">}}区域{{</hover>}}键的字符串值。

```yaml {label="xr"}
apiVersion: custom-api.example.org/v1alpha1
kind: xDatabase
metadata:
  name: my-composite-resource
spec: 
  region: "US"
```

{{<hint "tip" >}}内定义的自定义应用程序接口{{<hover label="schema" line="18">}}spec.properties{{</hover>}}是 OpenAPIv3 规范。Swagger 文档的 [data models page](https://swagger.io/docs/specification/data-models/) 提供了使用数据类型和输入限制的示例列表。

Kubernetes 文档列出了关于 OpenAPIv3 自定义 API 可以使用的 [一组特殊限制](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation)。{{< /hint >}}

{{<hint "important" >}}

更改或扩展 XRD 模式需要重新启动 [Crossplane pod]({{<ref "./pods#crossplane-pod">}}) 才能生效。{{< /hint >}}

##### 必填字段

默认情况下，模式中的所有字段都是可选的。{{< hover label="required" line="25">}}属性{{</hover>}}属性将参数定义为必填项。

在本例中，X 射线衍射需要{{< hover label="required" line="19">}}区域{{</hover>}}和{{< hover label="required" line="21">}}尺寸{{</hover>}}但{{< hover label="required" line="23">}}名称{{</hover>}}是可选的。

```yaml {label="required",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string 
              size:
                type: string  
              name:
                type: string  
            required: 
              - region 
              - size
    # Removed for brevity
```

根据 OpenAPIv3 规范，"必要 "字段是按对象设置的。 如果模式包含多个对象，模式可能需要多个 "必要 "字段。

该 XRD 定义了两个对象: 

1. 顶级 {{<hover label="required2" line="7">}}对象{{</hover>}}对象
2. 第二个 {{<hover label="required2" line="14">}}对象{{</hover>}}对象

规格 {{<hover label="required2" line="7">}}规格{{</hover>}}对象{{<hover label="required2" line="23">}}要求{{</hover>}}的{{<hover label="required2" line="10">}}大小{{</hover>}}和{{<hover label="required2" line="14">}}位置{{</hover>}}但{{<hover label="required2" line="12">}}名称{{</hover>}}是可选的。

所需的 {{<hover label="required2" line="14">}}位置{{</hover>}}对象、{{<hover label="required2" line="17">}}国家{{</hover>}}是{{<hover label="required2" line="21">}}是{{</hover>}}和{{<hover label="required2" line="19">}}区{{</hover>}}为可选项。

```yaml {copy-lines="none",label="required2"}
# Removed for brevity
- name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string  
              name:
                type: string 
              location:
                type: object
                properties:
                  country: 
                    type: string 
                  zone:
                    type: string
                required:
                  - country
            required:  
              - size
              - location
```

Swagger "[描述参数](https://swagger.io/docs/specification/describing-parameters/) "文档中有更多示例。

##### crossplane 保留字段

crossplane 不允许在模式中使用以下字段: 

* `spec.resourceRef` 参数
* `spec.resourceRefs`参数
* `spec.claimRef` `spec.writeConnectionSecretToRef`.
* `spec.writeConnectionSecretToRef`.
* 状态.条件
* 状态.连接细节

crossplane 会忽略任何与保留字段匹配的字段。

#### 服务和引用模式

被引用的模式必须是{{<hover label="served" line="12" >}}服务: true{{</hover >}}和{{<hover label="served" line="13" >}}可引用: true{{</hover>}}.

```yaml {label="served"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string
```

Composition 资源可以使用任何被引用为{{<hover label="served" line="12" >}}服务: true。{{</hover >}}Kubernetes会拒绝任何被引用为 "served: false "的模式版本的复合资源。

{{< hint "tip" >}}将模式版本设置为 "served:false "会导致使用旧模式的用户出错。 在删除旧模式版本之前，这可以有效地识别和升级用户。{{< /hint >}}

可参考:  true {{<hover label="served" line="13" >}}referenceable: true{{</hover>}}字段表示 Composition 使用模式的哪个版本。 只能有一个版本是 "可引用的"。

{{< hint "note" >}}更改哪个版本为 `referenceable:true` 需要[更新 `compositeTypeRef.apiVersion`]({{<ref "./compositions#enabling-composite-resources" >}})。{{< /hint >}}

#### 多个模式版本

{{<hint "warning" >}}crossplane 支持定义多个 "版本"，但每个版本的模式不能改变任何现有字段，也称为 "进行破坏性更改"。

要打破不同版本之间的模式变化，需要使用[转换 webhooks](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion)。

新版本可以定义新的可选参数，但新的必填字段属于 "重大变更"。

<!-- vale Crossplane.Spelling = NO -->

<!-- ignore to allow for CRDs -->

<!-- don't add to the spelling exceptions to catch when it's used instead of XRD -->

crossplane XRD 使用 Kubernetes 自定义资源定义来进行版本控制。请阅读 Kubernetes 文档 [versions in CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)，了解有关版本和中断更改的更多背景信息。

<!-- vale Crossplane.Spelling = YES -->

crossplane 建议将模式更改作为全新的 XRD 来实施。{{< /hint >}}

对于 XRD，要创建新版本的应用程序接口，可添加一个新的{{<hover label="ver" line="21">}}名称{{</hover>}}在{{<hover label="ver" line="10">}}版本{{</hover>}}列表中添加新名称。

例如，该 XRD 版本{{<hover label="ver" line="11">}}v1alpha1{{</hover>}}只有字段{{<hover label="ver" line="19">}}区域{{</hover>}}.

第二个版本{{<hover label="ver" line="21">}}v1{{</hover>}}扩展了应用程序接口，使{{<hover label="ver" line="29">}}区域{{</hover>}}和{{<hover label="ver" line="31">}}大小{{</hover>}}.

```yaml {label="ver",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string  
  - name: v1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string 
              size:
                type: string
```

{{<hint "important" >}}

更改或扩展 XRD 模式需要重新启动 [Crossplane pod]({{<ref "./pods#crossplane-pod">}}) 才能生效。{{< /hint >}}

### 启用claim

作为选项，XRD 可以允许 claims 使用 XRD API。

{{<hint "note" >}}

请阅读 [Claims]({{<ref "./claims">}}) 页面获取更多有关claim的信息。{{</hint >}}

XRD 为claim提供了一个{{<hover label="claim" line="10">}}claim名称{{</hover >}}对象的claim。

claim名称 {{<hover label="claim" line="10">}}名称{{</hover >}}定义了{{<hover label="claim" line="11">}}种类{{</hover >}}和{{<hover label="claim" line="12">}}复数{{</hover >}}就像 XRD{{<hover label="claim" line="7">}}名称{{</hover >}}也像 XRD{{<hover label="claim" line="7">}}名称{{</hover >}}一样，被引用的{{<hover label="claim" line="11">}}种类{{</hover >}}使用大写，而{{<hover label="claim" line="12">}}复数{{</hover >}}.

claims{{<hover label="claim" line="11">}}种类{{</hover >}}和{{<hover label="claim" line="12">}}复数{{</hover >}}必须是唯一的，不能与任何其他claim或其他 XRD{{<hover label="claim" line="8">}}种类{{</hover >}}.

{{<hint "tip" >}}常见的 crossplane 惯例是使用{{<hover label="claim" line="10">}}claim名称{{</hover >}}与 XRD{{<hover label="claim" line="7">}}名称{{</hover >}}但不以 "x "开头。{{</hint >}}

```yaml {label="claim",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  versions:
  # Removed for brevity
```

{{<hint "important" >}}您不能更改{{<hover label="claim" line="10">}}claim名称{{</hover >}}您必须删除并重新创建 XRD 才能更改{{<hover label="claim" line="10">}}claim名称{{</hover >}}.{{</hint >}}

#### 管理连接secret

当复合资源创建托管资源时，Crossplane 会提供任何 [connection secrets]({{<ref "./managed-resources#writeconnectionsecrettoref">}}) 提供给复合资源或 claims。 这就要求复合资源和 claims 的创建者知道受管资源提供的secret。 在其他情况下，Crossplane 管理员可能不想公开部分或全部生成的连接secret。

XRD 可以定义一个{{<hover label="key" line="10">}}连接秘钥{{</hover>}}来限制提供给 Compider 资源或 Claim 的内容。

crossplane 只提供连接秘钥（connectionSecretKeys）中列出的密钥。{{<hover label="key" line="10">}}中列出的密钥。{{</hover>}}中列出的密钥，其他任何连接secret都不会被引用到复合资源或 claims。

{{<hint "important" >}}中列出的密钥{{<hover label="key" line="10">}}连接密钥{{</hover>}}中列出的密钥名称必须与 Composition 的 "connectionDetails "中列出的密钥名称一致。

XRD 会忽略列出的任何非托管资源创建的键。

更多信息，请阅读 [Composition documentation]({{<ref "./compositions#storing-connection-details">}}).{{< /hint >}}

例如，XRD 传递的键是{{<hover label="key" line="11">}}用户名{{</hover>}},{{<hover label="key" line="12">}}密码{{</hover>}}和{{<hover label="key" line="13">}}地址{{</hover>}}.

Composition 资源或 claims 会将这些内容保存在由其 `writeConnectionSecretToRef` 字段定义的 secret 中。

```yaml {label="key",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  connectionSecretKeys:
    - username
    - password
    - address
  versions:
  # Removed for brevity
```

{{<hint "warning">}}您不能更改 XRD 的 "connectionSecretKeys"，必须删除并重新创建 XRD 才能更改 "connectionSecretKeys"。{{</hint >}}

有关连接secret的更多信息，请阅读[连接secret知识库文章]({{<ref "/knowledge-base/guides/connection-details">}}).

### 设置 Composition 资源默认值

XRD 可以为 Composition 资源和claim设置默认参数。

<!-- vale off -->

#### defaultCompositeDeletePolicy

<!-- vale on -->

如果用户在创建claim时没有指定值，"defaultCompositeDeletePolicy "会定义claim的 "compositeDeletePolicy "属性的默认值。 claim控制器会使用 "compositeDeletePolicy "属性来指定删除关联composition时的传播策略。 compositeDeletePolicy "不适用于没有关联claim的独立composition。

使用 "defaultCompositeDeletePolicy: Background "策略会使claim的 CRD 的 "compositeDeletePolicy "属性的默认值为 "Background"。 当删除的claim的 "compositeDeletePolicy "属性设置为 "Background "时，claim控制器会使用传播策略 "background "删除复合资源并返回，依靠 Kubernetes 删除剩余的子对象，如托管资源、嵌套的复合资源和secret。

使用 `defaultCompositeDeletePolicy: Foreground` 会使claim的 CRD 具有 `compositeDeletePolicy` 的默认值 `Foreground`。 当删除的claim的 `compositeDeletePolicy` 属性设置为 `Foreground`时，控制器会使用传播策略 `foreground` 删除相关的 Composition。 这会导致 Kubernetes 使用前台级联删除，即在删除父资源之前删除所有子资源。 claim控制器会等待 Composition 删除完成后再返回。

在创建claim时，用户可以通过包含具有 "Background "或 "Foreground "值的 "spec.compositeDeletePolicy "属性来覆盖 "defaultCompositeDeletePolicy"。

默认值为 `defaultCompositeDeletePolicy: Background`。

设置{{<hover label="delete" line="6">}}defaultCompositeDeletePolicy: Foreground{{</hover>}}来更改 XRD 删除策略。

```yaml {label="delete",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositeDeletePolicy: Foreground
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->

#### defaultCompositionRef

<!-- vale on -->

多个 [Composition]({{<ref "./compositions">}}如果有多个 Composition 引用同一个 XRD，复合资源或 claims 必须选择使用哪个 Composition。

XRD 可以使用 `defaultCompositionRef` 值定义要使用的默认 Composition。

设置一个{{<hover label="defaultComp" line="6">}}defaultCompositionRef{{</hover>}}以设置默认 Composition。

```yaml {label="defaultComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositionRef: 
    name: myComposition
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->

#### defaultCompositionUpdatePolicy

<!-- vale on -->

对 Composition 的更改会生成新的 Composition 修订版。 默认情况下，所有复合资源和claim都会被引用更新后的 Composition 修订版。

将 XRD 的 "defaultCompositionUpdatePolicy "设置为 "Manual"，以防止composition资源和 claims 自动被引用新修订。

默认值为 `defaultCompositionUpdatePolicy: Automatic`。

设置 {{<hover label="compRev" line="6">}}defaultCompositionUpdatePolicy: Manual{{</hover>}}可为被引用此 XRD 的复合资源和 claims 设置默认的 Composition 更新策略。

```yaml {label="compRev",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositionUpdatePolicy: Manual
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->

#### enforcedCompositionRef

<!-- vale on -->

要要求所有复合资源或 claims 使用特定 Composition，请使用 XRD 中的 "enforcedCompositionRef "设置。

例如，要求使用该 XRD 的所有复合资源和claim都使用 Composition{{<hover label="enforceComp" line="6">}}我的composition{{</hover>}}设置{{<hover label="enforceComp" line="6">}}enforcedCompositionRef.name: myComposition{{</hover>}}.

```yaml {label="defaultComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  enforcedCompositionRef: 
    name: myComposition
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

## 验证复合资源定义

使用 `kubectl get compositeresourcedefinition` 或简写形式验证 XRD、{{<hover label="getxrd" line="1">}}kubectl get xrd{{</hover>}}.

```yaml {label="getxrd",copy-lines="1"}
kubectl get xrd                                
NAME ESTABLISHED OFFERED AGE
xdatabases.custom-api.example.org True True 22m
```

ESTABLISHED "字段表示 crossplane 为该 XRD 安装了 Kubernetes 自定义资源定义。

OFFERED "字段表示该 XRD 提供了一个 claims，并且 crossplane 为该 Claim 安装了 Kubernetes 自定义资源定义。

### XRD 条件

crossplane 对 XRD 使用一套标准的 "Conditions"（条件）。 使用 "kubectl describe xrd "查看 XRD 的 "Status"（状态）下的条件。

```yaml {copy-lines="none"}
kubectl describe xrd
Name:         xpostgresqlinstances.database.starter.org
API Version:  apiextensions.crossplane.io/v1
Kind:         CompositeResourceDefinition
# Removed for brevity
Status:
  Conditions:
    Reason:                WatchingCompositeResource
    Status:                True
    Type:                  Established
    Reason:                WatchingCompositeResourceClaim
    Status:                True
    Type:                  Offered
# Removed for brevity
```

<!-- vale off -->

#### WatchingCompositeResource

<!-- vale on -->

`Reason: WatchingCompositeResource` 表示 crossplane 定义了与 Composite 资源相关的新 Kubernetes 自定义资源定义，并正在观察新 Composite 资源的创建。

```yaml
Type: Established
Status: True
Reason: WatchingCompositeResource
```

<!-- vale off -->

#### 终止复合资源

<!-- vale on -->

Reason: TerminatingCompositeResource "表示 crossplane 正在删除与 Composite 资源相关的自定义资源定义，并正在终止 Composite 资源控制器。

```yaml
Type: Established
Status: False
Reason: TerminatingCompositeResource
```

<!-- vale off -->

#### WatchingCompositeResourceClaim

<!-- vale on -->

`Reason: WatchingCompositeResourceClaim` 表示 crossplane 定义了与所提供的 Claims 相关的新 Kubernetes 自定义资源定义，并正在观察新 Claims 的创建。

```yaml
Type: Offered
Status: True
Reason: WatchingCompositeResourceClaim
```

<!-- vale off -->

#### TerminatingCompositeResourceClaim

<!-- vale on -->

Reason: TerminatingCompositeResourceClaim "表示 crossplane 正在删除与提供的claim相关的自定义资源定义，并终止claim控制器。

```yaml
Type: Offered
Status: False
Reason: TerminatingCompositeResourceClaim
```
