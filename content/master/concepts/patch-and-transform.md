---

title: 补丁和变换
weight: 70
description: "在创建受管资源之前，Crossplane Composition 使用补丁和变换来修改来自claim和复合资源的输入"

---

Crossplane Composition 允许进行 "修补和转换 "操作。 通过修补，Composition 可以对 Composition 所定义的资源应用更改。

当用户创建 Composition 时，crossplane 会将 Composition 中的设置传递给关联的复合资源。 补丁可以使用这些设置来更改关联的复合资源或受管资源。

被引用补丁和变换的例子包括

* 更改外部资源的名称
* 将 "东 "或 "西 "等通用术语映射到特定的 Providers 位置
* 为资源字段添加自定义标签或字符串

{{<hint "note" >}}

<!-- vale alex.Condescending = NO -->

Crossplane 希望修补和转换操作是简单的修改。 使用 [Composition Functions]({{<ref "./composition-functions">}}) 进行更复杂或程序化的修改。

<!-- vale alex.Condescending = YES -->

{{</hint >}}

Composition [patch](#create-a-patch) 是更改字段的操作。 Composition [transform](#transform-a-patch) 是在应用补丁之前修改数值。

## 创建补丁

补丁是单个{{<hover label="createComp" line="4">}}资源{{</hover>}}内的{{<hover label="createComp" line="2">}}Composition{{</hover>}}.

补丁 {{<hover label="createComp" line="8">}}补丁{{</hover>}}字段包含一个要应用到单个资源的补丁列表。

每个补丁都有一个 {{<hover label="createComp" line="9">}}类型{{</hover>}}，它定义了 crossplane 应用何种补丁操作。

根据补丁类型的不同，补丁引用复合资源或 Composition 内部字段的方式也不同，但所有补丁都会引用一个{{<hover label="createComp" line="10">}}字段路径{{</hover>}}和{{<hover label="createComp" line="11">}}字段路径{{</hover>}}.

字段路径 {{<hover label="createComp" line="10">}}字段路径{{</hover>}}定义补丁的输入值。 {{<hover label="createComp" line="11">}}字段路径{{</hover>}}定义要通过补丁更改的数据。

下面是一个应用于 Composition 中资源的补丁示例。

```yaml {label="createComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: my-composed-resource
      base:
        # Removed for brevity
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.field1
          toFieldPath: metadata.labels["patchLabel"]
```

### 选择字段

crossplane 使用[JSONPath 选择器](https://kubernetes.io/docs/reference/kubectl/jsonpath/)的一个子集（称为 "字段选择器"）来选择 Composition 资源或 managed 资源中的字段。

字段选择器可以选择 Composition 资源或托管资源对象中的任何字段，包括 "metadata"、"spec "或 "status "字段。

字段选择符可以是与字段名或数组索引相匹配的字符串，放在括号中。 字段名可以使用". "字符来选择子元素。

#### 字段选择器示例

下面是一些来自 Composition 资源对象的选择器示例。{{<table "table" >}}| 选择器 | 被选中的元素 | | --- | --- | | `kind` | {{<hover label="select" line="3">}}种类{{</hover>}}| | `metadata.labels['crossplane.io/claim-name']` | | `metadata.labelels['crossplane.io/claim-name']` {{<hover label="select" line="7">}}我的示例claim{{</hover>}}| | `spec.desiredRegion` | | `spec.desiredRegion`. {{<hover label="select" line="11">}}eu-north-1{{</hover>}}| `spec.resourceRefs[0].name` | `spec.resourceRefs[0].name` | `spec.resourceRefs[0].name {{<hover label="select" line="16">}}my-example-claim-978mh-r6z64{{</hover>}}|{{</table >}}

```yaml {label="select",copy-lines="none"}
$ kubectl get composite -o yaml
apiVersion: example.org/v1alpha1
kind: xExample
metadata:
  # Removed for brevity
  labels:
    crossplane.io/claim-name: my-example-claim
    crossplane.io/claim-namespace: default
    crossplane.io/composite: my-example-claim-978mh
spec:
  desiredRegion: eu-north-1
  field1: field1-text
  resourceRefs:
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-r6z64
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-cnlhj
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-rv5nm
  # Removed for brevity
```

## 重复使用补丁

Composition 可以通过 PatchSet 在多个资源上重复使用一个补丁对象。

要创建补丁集，请定义一个{{<hover label="patchset" line="5">}}补丁集{{</hover>}}对象。{{<hover label="patchset" line="4">}}对象{{</hover>}}.

PatchSet 中的每个补丁都有一个{{<hover label="patchset" line="6">}}名称{{</hover>}}和一个{{<hover label="patchset" line="7">}}补丁{{</hover>}}.

{{<hint "note" >}}对于多个补丁集，只需被引用一个{{<hover label="patchset" line="5">}}补丁集{{</hover>}}对象。

用一个唯一的{{<hover label="patchset" line="6">}}名称{{</hover>}}.{{</hint >}}

将 PatchSet 应用于带有补丁的资源{{<hover label="patchset" line="16">}}类型: 补丁集。{{</hover>}}设置{{<hover label="patchset" line="17">}}patchSetName{{</hover>}}设置为{{<hover label="patchset" line="6">}}的{{</hover>}}的名称。

```yaml {label="patchset"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
spec:
  patchSets:
  - name: my-patchset
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.desiredRegion
      toFieldPath: spec.forProvider.region
  resources:
    - name: bucket1
      base:
        # Removed for brevity
      patches:
        - type: PatchSet
          patchSetName: my-patchset
    - name: bucket2
      base:
        # Removed for brevity
      patches:
        - type: PatchSet
          patchSetName: my-patchset
```

{{<hint "important" >}}补丁集不能包含其他补丁集。

crossplane 会忽略 PatchSet 中的任何 [transforms](#transform-a-patch) 或 [policies](#patch-policies) 。{{< /hint >}}

## 资源之间的修补

Composition 无法直接在同一 Composition 中的资源之间进行修补。 例如，生成网络资源并将资源名称修补为计算资源。

{{<hint "important">}}ToEnvironmentFieldPath](#toenvironmentfieldpath) 补丁无法从 `Status` 字段读取数据。{{< /hint >}}

资源可以修补到用户定义的{{<hover label="xrdPatch" line="13">}}状态{{</hover>}}字段。

然后，资源可以从该{{<hover label="xrdPatch" line="13">}}状态{{</hover>}}字段来修补一个字段。

首先，定义一个自定义{{<hover label="xrdPatch" line="13">}}状态{{</hover>}}和自定义字段，例如{{<hover label="xrdPatch" line="16">}}第二个资源{{</hover>}}

```yaml {label="xrdPatch",copy-lines="13-17"}
kind: CompositeResourceDefinition
# Removed for brevity.
spec:
  # Removed for brevity.
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            # Removed for brevity.
          status:
              type: object
              properties:
                secondResource:
                  type: string
```

在 Composition 内部，带有源数据的资源被引用一个{{<hover label="patchBetween" line="10">}}到组合字段路径{{</hover>}}补丁将数据写入{{<hover label="patchBetween" line="12">}}status.secondResource{{</hover>}}字段。

目标资源被引用为{{<hover label="patchBetween" line="19">}}从复合字段路径{{</hover>}}补丁从 Composition 资源中读取数据{{<hover label="patchBetween" line="20">}}status.secondResource{{</hover>}}字段中的数据，并将其写入名为{{<hover label="patchBetween" line="21">}}标签中。{{</hover>}}的标签。

```yaml {label="patchBetween",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        # Removed for brevity
      patches:
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: status.secondResource
    - name: bucket2
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        # Removed for brevity
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: status.secondResource
          toFieldPath: metadata.labels['secondResource']
```

描述 Composition 资源以查看{{<hover label="descCompPatch" line="5">}}资源{{</hover>}}和{{<hover label="descCompPatch" line="11">}}status.secondResource{{</hover>}}Values 值。

```yaml {label="descCompPatch",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-jp7rx
Spec:
  # Removed for brevity
  Resource Refs:
    Name:         my-example-claim-jp7rx-gfg4m
    # Removed for brevity
    Name:         my-example-claim-jp7rx-fttpj
Status:
  # Removed for brevity
  Second Resource:         my-example-claim-jp7rx-gfg4m
```

描述要查看标签的目标托管资源{{<hover label="bucketlabel" line="5">}}第二个资源{{</hover>}}.

```yaml {label="bucketlabel",copy-lines="none"}
$ kubectl describe bucket
kubectl describe bucket my-example-claim-jp7rx-fttpj
Name:         my-example-claim-jp7rx-fttpj
Labels:       crossplane.io/composite=my-example-claim-jp7rx
              secondResource=my-example-claim-jp7rx-gfg4m
```

## 补丁类型

crossplane 支持多种补丁类型，每种类型都被引用不同的数据源，并将补丁应用到不同的位置。

{{<hint "important" >}}

本节介绍应用于 Composition 内部单个资源的补丁。

有关使用 Composition 的 `environment.patches` 对整个复合资源应用补丁的信息，请阅读 [Environment Configurations]({{<ref "environment-configs" >}}) 文档。

{{< /hint >}}

crossplane 修补程序摘要{{< table "table table-hover" >}}| Patch Type | Data Source | Data Destination | 
| ---  | --- | --- | 
| [FromCompositeFieldPath](#fromcompositefieldpath) | A field in the composite resource. | A field in the patched managed resource. | 
| [ToCompositeFieldPath](#tocompositefieldpath) | A field in the patched managed resource. | A field in the composite resource. |  
| [CombineFromComposite](#combinefromcomposite) | Multiple fields in the composite resource. | A field in the patched managed resource. | 
| [CombineToComposite](#combinetocomposite) | Multiple fields in the patched managed resource. | A field in the composite resource. | 
| [FromEnvironmentFieldPath](#fromenvironmentfieldpath) | Data in the in-memory EnvironmentConfig Environment | A field in the patched managed resource. | 
| [ToEnvironmentFieldPath](#toenvironmentfieldpath) | A field in the patched managed resource. | The in-memory EnvironmentConfig Environment. | 
| [CombineFromEnvironment](#combinefromenvironment) | Multiple fields in the in-memory EnvironmentConfig Enviro{{< /table >}}

{{<hint "note" >}}以下所有示例都使用了同一套 Composition、CompositeResourceDefinitions、Claims 和 EnvironmentConfigs。 不同示例中只有被引用的补丁有所变化。

所有示例都依赖 upbound [Provider-aws-s3](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/) 来创建资源。

{{< expand "Reference Composition" >}}

```yaml {copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: xExample
  environment:
    environmentConfigs:
    - ref:
        name: example-environment
  resources:
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
    - name: bucket2
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
```

{{< /expand >}}

{{<expand "Reference CompositeResourceDefinition" >}}

```yaml {copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xexamples.example.org
spec:
  group: example.org
  names:
    kind: xExample
    plural: xexamples
  claimNames:
    kind: ExampleClaim
    plural: exampleclaims
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
              field1:
                type: string
              field2:
                type: string
              field3: 
                type: string
              desiredRegion: 
                type: string
              boolField:
                type: boolean
              numberField:
                type: integer
          status:
              type: object
              properties:
                url:
                  type: string
```

{{< /expand >}}

{{< expand "Reference Claim" >}}

```yaml {copy-lines="all"}
apiVersion: example.org/v1alpha1
kind: ExampleClaim
metadata:
  name: my-example-claim
spec:
  field1: "field1-text"
  field2: "field2-text"
  desiredRegion: "eu-north-1"
  boolField: false
  numberField: 10
```

{{< /expand >}}

{{< expand "Reference EnvironmentConfig" >}}

```yaml {copy-lines="all"}
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
```

{{< /expand >}}
{{< /hint >}}

<!-- vale Google.Headings = NO -->

### FromCompositeFieldPath

<!-- vale Google.Headings = YES -->

从{{<hover label="fromComposite" line="12">}}字段路径{{</hover>}}补丁获取 Composition 资源中的 Values 并将其应用到托管资源中的字段。

{{< hint "tip" >}}被引用{{<hover label="fromComposite" line="12">}}FromCompositeFieldPath{{</hover>}}补丁将用户在其 claims 中的选项应用到 managed resource `forProvider` 设置中。{{< /hint >}}

例如，要被引用的值是{{<hover label="fromComposite" line="13">}}desiredRegion{{</hover>}}值到托管资源的{{<hover label="fromComposite" line="10">}}区域{{</hover>}}.

字段路径 {{<hover label="fromComposite" line="13">}}字段路径{{</hover>}}值是 Composition 资源中的一个字段。

到字段路径 {{<hover label="fromComposite" line="14">}}字段路径{{</hover>}}值是要更改的托管资源中的字段。

```yaml {label="fromComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.desiredRegion
          toFieldPath: spec.forProvider.region
```

查看托管资源，查看更新后的{{<hover label="fromCompMR" line="6">}}区域{{</hover>}}

```yaml {label="fromCompMR",copy-lines="1"}
$ kubectl describe bucket
Name:         my-example-claim-qlr68-29nqf
# Removed for brevity
Spec:
  For Provider:
    Region:  eu-north-1
```

<!-- vale Google.Headings = NO -->

### ToCompositeFieldPath

<!-- vale Google.Headings = YES -->

到{{<hover label="toComposite" line="12">}}ToCompositeFieldPath{{</hover>}}将数据从单个托管资源写入创建它的 Composition 资源。

{{< hint "tip" >}}被引用 {{<hover label="toComposite" line="12">}}到组合字段路径{{</hover>}}补丁从 Composition 中的一个托管资源中获取数据，并将其引用到同一 Composition 中的第二个托管资源中。{{< /hint >}}

例如，在 crossplane 创建新的托管资源后，取值为{{<hover label="toComposite" line="13">}}托管区 ID{{</hover>}}作为{{<hover label="toComposite" line="14">}}标签{{</hover>}}作为标签应用到 Composition 资源中。

```yaml {label="toComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.hostedZoneId
          toFieldPath: metadata.labels['ZoneID']
```

查看已创建的托管资源，查看{{<hover label="toCompMR" line="6">}}托管区域 ID{{</hover>}}字段。

```yaml {label="toCompMR",copy-lines="none"}
$ kubectl describe bucket
Name:         my-example-claim-p5pxf-5vnp8
# Removed for brevity
Status:
  At Provider:
    Hosted Zone Id:       Z2O1EMRO9K5GLX
    # Removed for brevity
```

接下来查看 Composition 资源，确认补丁应用了{{<hover label="toCompositeXR" line="3">}}标签{{</hover>}}

```yaml {label="toCompositeXR",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-p5pxf
Labels:       ZoneID=Z2O1EMRO9K5GLX
```

{{<hint "important">}}Crossplane 在创建托管资源后，直到下一个对账循环才会将修补程序应用到复合资源。 这就造成了托管资源就绪与应用修补程序之间的延迟。{{< /hint >}}

<!-- vale Google.Headings = NO -->

### CombineFromComposite

<!-- vale Google.Headings = YES -->

组合{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}}补丁从 Composition 资源中获取 Values，将其组合并应用到托管资源。

{{< hint "tip" >}}被引用{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}}补丁来创建复杂的字符串（如安全策略），并将其应用到托管资源。{{< /hint >}}

例如，被引用的 claims 值为{{<hover label="combineFromComp" line="15">}}desiredRegion{{</hover>}}和{{<hover label="combineFromComp" line="16">}}字段2{{</hover>}}来生成托管资源的{{<hover label="combineFromComp" line="20">}}名称{{</hover>}}

组合{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}}补丁仅支持{{<hover label="combineFromComp" line="13">}}组合{{</hover>}}选项。

变量 {{<hover label="combineFromComp" line="14">}}变量{{</hover>}}是{{<hover label="combineFromComp" line="15">}}字段路径{{</hover>}}值的列表。

唯一支持的{{<hover label="combineFromComp" line="17">}}策略{{</hover>}}是{{<hover label="combineFromComp" line="17">}}策略: 字符串{{</hover>}}.

可选择应用{{<hover label="combineFromComp" line="19">}}string.fmt{{</hover>}}来指定如何组合字符串。

到字段路径 {{<hover label="combineFromComp" line="20">}}字段路径{{</hover>}}是要应用新字符串的托管资源中的字段。

```yaml {label="combineFromComp",copy-lines="11-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: spec.desiredRegion
              - fromFieldPath: spec.field2
            strategy: string
            string:
              fmt: "my-resource-%s-%s"
          toFieldPath: metadata.name
```

描述受管资源，查看应用的补丁。

```yaml {label="describeCombineFromComp",copy-lines="none"}
$ kubectl describe bucket
Name:         my-resource-eu-north-1-field2-text
```

<!-- vale Google.Headings = NO -->

### CombineToComposite

<!-- vale Google.Headings = YES -->

组合{{<hover label="combineToComposite" line="12">}}组合到复合{{</hover>}}补丁从托管资源中获取 Values，将其组合并应用于 Composition 资源。

{{<hint "tip" >}}被引用 {{<hover label="combineToComposite" line="12">}}CombineToComposite{{</hover>}}补丁从托管资源中的多个字段创建类似 URL 的单个字段。{{< /hint >}}

例如，被引用的 managed resource{{<hover label="combineToComposite" line="15">}}名称{{</hover>}}和{{<hover label="combineToComposite" line="16">}}区域{{</hover>}}生成自定义{{<hover label="combineToComposite" line="20">}}url{{</hover>}}字段。

{{< hint "important" >}}在 Composition 资源的 Status 字段中写入自定义字段需要先在 CompositeResourceDefinition 中定义自定义字段。

{{< /hint >}}

组合{{<hover label="combineToComposite" line="12">}}组合到composition{{</hover>}}补丁仅支持{{<hover label="combineToComposite" line="13">}}组合{{</hover>}}选项。

变量 {{<hover label="combineToComposite" line="14">}}变量{{</hover>}}是{{<hover label="combineToComposite" line="15">}}fromFieldPath{{</hover>}}要合并的 managed 资源。

唯一支持的{{<hover label="combineToComposite" line="17">}}策略{{</hover>}}是{{<hover label="combineToComposite" line="17">}}策略: 字符串{{</hover>}}.

可选择应用{{<hover label="combineToComposite" line="19">}}string.fmt{{</hover>}}来指定如何组合字符串。

到字段路径 {{<hover label="combineToComposite" line="20">}}字段路径{{</hover>}}是要应用新字符串的 Composition 资源中的字段。

```yaml {label="combineToComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineToComposite
          combine:
            variables:
              - fromFieldPath: metadata.name
              - fromFieldPath: spec.forProvider.region
            strategy: string
            string:
              fmt: "https://%s.%s.com"
          toFieldPath: status.url
```

查看 Composition 资源以验证应用的补丁。

```yaml {copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-bjdjw
API Version:  example.org/v1alpha1
Kind:         xExample
# Removed for brevity
Status:
  # Removed for brevity
  URL:                     https://my-example-claim-bjdjw-r6ncd.us-east-2.com
```

<!-- vale Google.Headings = NO -->

### FromEnvironmentFieldPath

<!-- vale Google.Headings = YES -->

{{<hint "important" >}}EnvironmentConfigs 是 alpha 功能，默认情况下不会启用。

有关使用 EnvironmentConfig 的更多信息，请阅读 [EnvironmentConfigs]({{<ref "./environment-configs">}}) 文档。{{< /hint >}}

从环境字段路径{{<hover label="fromEnvField" line="12">}}从环境字段路径{{</hover>}}补丁从内存中的 EnvironmentConfig 环境中获取 Values 并将其应用到托管资源。

{{<hint "tip" >}}被引用{{<hover label="fromEnvField" line="12">}}从环境字段路径{{</hover>}}来应用基于当前环境的自定义托管资源设置。{{< /hint >}}

例如，被引用环境的{{<hover label="fromEnvField" line="13">}}locations.eu{{</hover>}}值，并将其应用为{{<hover label="fromEnvField" line="14">}}区域{{</hover>}}.

```yaml {label="fromEnvField",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
        patches:
        - type: FromEnvironmentFieldPath
          fromFieldPath: locations.eu
          toFieldPath: spec.forProvider.region
```

验证托管资源，确认已应用补丁。

```yaml {copy-lines="none"}
kubectl describe bucket
Name:         my-example-claim-8vrvc-xx5sr
Labels:       crossplane.io/claim-name=my-example-claim
# Removed for brevity
Spec:
  For Provider:
    Region:  eu-north-1
  # Removed for brevity
```

<!-- vale Google.Headings = NO -->

### ToEnvironmentFieldPath

<!-- vale Google.Headings = YES -->

{{<hint "important" >}}EnvironmentConfigs 是 alpha 功能，默认情况下不会启用。

有关使用 EnvironmentConfig 的更多信息，请阅读 [EnvironmentConfigs]({{<ref "./environment-configs">}}) 文档。{{< /hint >}}

环境字段路径{{<hover label="toEnvField" line="12">}}环境字段路径{{</hover>}}补丁获取托管资源的 Values 并将其应用到内存中的 EnvironmentConfig 环境。

{{<hint "tip" >}}被引用{{<hover label="toEnvField" line="12">}}ToEnvironmentFieldPath{{</hover>}}将数据写入任何 FromEnvironmentFieldPath 补丁都能访问的环境。{{< /hint >}}

例如，被引用的{{<hover label="toEnvField" line="13">}}区域{{</hover>}}值，并将其应用为环境的{{<hover label="toEnvField" line="14">}}键1{{</hover>}}.

```yaml {label="toEnvField",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
        patches:
        - type: ToEnvironmentFieldPath
          fromFieldPath: spec.forProvider.region
          toFieldPath: key1
```

由于环境是内存中的，因此没有命令来确认补丁是否将值写入了环境。

{{<hint "important">}}环境字段路径{{<hover label="toEnvField" line="12">}}环境字段路径{{</hover>}}补丁发生在***创建资源之前。{{<hover label="toEnvField" line="13">}}字段路径{{</hover>}}无法读取 `atProvider` 或 `Status` 字段。{{< /hint >}}

<!-- vale Google.Headings = NO -->

### CombineFromEnvironment

<!-- vale Google.Headings = YES -->

{{<hint "important" >}}EnvironmentConfigs 是 alpha 功能，默认情况下不会启用。

有关使用 EnvironmentConfig 的更多信息，请阅读 [EnvironmentConfigs]({{<ref "./environment-configs">}}) 文档。{{< /hint >}}

从环境组合{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}补丁结合了来自内存中 EnvironmentConfig 环境的多个 Values，并将其应用于托管资源。

{{<hint "tip" >}}被引用{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}补丁来创建复杂的字符串（如安全策略），并将其应用到受管资源。{{< /hint >}}

例如，将环境中的多个字段组合起来，创建一个唯一的{{<hover label="combineFromEnv" line="20">}}Annotations{{</hover>}}.

从环境组合{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}补丁仅支持{{<hover label="combineFromEnv" line="13">}}结合{{</hover>}}选项。

唯一支持的{{<hover label="combineFromEnv" line="14">}}策略{{</hover>}}是{{<hover label="combineFromEnv" line="14">}}策略: 字符串{{</hover>}}.

变量 {{<hover label="combineFromEnv" line="15">}}变量{{</hover>}}是{{<hover label="combineFromEnv" line="16">}}字段路径{{</hover>}}值的列表。

可选择应用{{<hover label="combineFromEnv" line="19">}}string.fmt{{</hover>}}来指定如何组合字符串。

到字段路径 {{<hover label="combineFromEnv" line="20">}}字段路径{{</hover>}}是要应用新字符串的托管资源中的字段。

```yaml {label="combineFromEnv",copy-lines="11-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineFromEnvironment
          combine:
            strategy: string
            variables:
            - fromFieldPath: key1
            - fromFieldPath: key2
            string: 
              fmt: "%s-%s"
          toFieldPath: metadata.annotations[EnvironmentPatch]
```

描述托管资源，以查看新的{{<hover label="combineFromEnvDesc" line="4">}}Annotations{{</hover>}}.

```yaml {copy-lines="none",label="combineFromEnvDesc"}
$ kubectl describe bucket
Name:         my-example-claim-zmxdg-grl6p
# Removed for brevity
Annotations:  EnvironmentPatch: value1-value2
# Removed for brevity
```

<!-- vale Google.Headings = NO -->

### CombineToEnvironment

<!-- vale Google.Headings = YES -->

{{<hint "important" >}}EnvironmentConfigs 是 alpha 功能，默认情况下不会启用。

有关使用 EnvironmentConfig 的更多信息，请阅读 [EnvironmentConfigs]({{<ref "./environment-configs">}}) 文档。{{< /hint >}}

组合到环境{{<hover label="combineToEnv" line="12">}}组合到环境{{</hover>}}补丁结合了来自托管资源的多个 Values，并将它们应用到内存中的 EnvironmentConfig 环境。

{{<hint "tip" >}}被引用{{<hover label="combineToEnv" line="12">}}组合环境{{</hover>}}补丁来创建复杂的字符串，如在其他托管资源中被引用的安全策略。{{< /hint >}}

例如，结合托管资源中的多个字段创建一个唯一字符串，并将其存储在环境的{{<hover label="combineToEnv" line="20">}}键2{{</hover>}}值中。

字符串结合了托管资源{{<hover label="combineToEnv" line="16">}}种类{{</hover>}}和{{<hover label="combineToEnv" line="17">}}区域{{</hover>}}.

组合到环境{{<hover label="combineToEnv" line="12">}}结合到环境{{</hover>}}补丁仅支持{{<hover label="combineToEnv" line="13">}}结合{{</hover>}}选项。

唯一支持的{{<hover label="combineToEnv" line="14">}}策略{{</hover>}}是{{<hover label="combineToEnv" line="14">}}策略: 字符串{{</hover>}}.

变量 {{<hover label="combineToEnv" line="15">}}变量{{</hover>}}是{{<hover label="combineToEnv" line="16">}}fromFieldPath{{</hover>}}值的列表。

可选择应用{{<hover label="combineToEnv" line="19">}}string.fmt{{</hover>}}来指定如何组合字符串。

到字段路径 {{<hover label="combineToEnv" line="20">}}toFieldPath{{</hover>}}是要写入新字符串的环境键。

{{< hint "important" >}}环境密钥必须已经存在。 补丁不能创建新的环境密钥。{{< /hint >}}

```yaml {label="combineToEnv",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineToEnvironment
          combine:
            strategy: string
            variables:
            - fromFieldPath: kind
            - fromFieldPath: spec.forProvider.region
            string:
              fmt: "%s.%s"
          toFieldPath: key2
```

由于环境是内存中的，因此没有命令来确认补丁是否将值写入了环境。

## 转换补丁

在应用修补程序时，Crossplane 支持在将数据作为修补程序应用之前对其进行修改。 Crossplane 将此称为 "转换 "操作。

crossplane 变换摘要。{{< table "table table-hover" >}}| 转换类型 | 操作 | | --- | --- | | [convert](#convert-transforms) | 将输入数据类型转换为不同类型。 也称为 "铸造"。 | | [map](#map-transforms) | 根据特定输入选择特定输出。 | | [match](#match-transform) | 根据字符串或正则表达式选择特定输出。 | | [math](#math-transforms) | 对输入应用数学运算。 | | [string](#string-transforms) | 使用 [Go string formatting](https://pkg.go.dev/fmt) 改变输入字符串。{{< /table >}}

使用{{<hover label="transform1" line="15">}}变换{{</hover>}}字段直接对单个补丁应用变换。

A{{<hover label="transform1" line="15">}}转换{{</hover>}}需要一个{{<hover label="transform1" line="16">}}类型{{</hover>}}表示要进行的变换操作。

另一个变换字段与{{<hover label="transform1" line="16">}}类型{{</hover>}}在本例中与{{<hover label="transform1" line="17">}}映射{{</hover>}}.

其他字段取决于被引用的补丁类型。

此示例被引用为{{<hover label="transform1" line="16">}}类型: 映射{{</hover>}}转换，输入{{<hover label="transform1" line="13">}}spec.desiredRegion{{</hover>}}匹配到{{<hover label="transform1" line="18">}}我们{{</hover>}}或{{<hover label="transform1" line="19">}}欧盟{{</hover>}}并为{{<hover label="transform1" line="14">}}spec.forProvider.region{{</hover>}}Values 的值返回相应的 AWS 区域。

```yaml {label="transform1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.desiredRegion
          toFieldPath: spec.forProvider.region
          transforms:
            - type: map
              map:
                us: us-east-2
                eu: eu-north-1
```

### 转换变换

转换 {{<hover label="convert" line="6">}}转换{{</hover>}}转换类型可将输入数据类型转换为不同的数据类型。

{{< hint "tip" >}}某些 Provider API 要求字段是字符串，请使用{{<hover label="convert" line="7">}}转换{{</hover>}}类型将布尔或整数字段改为字符串。{{< /hint >}}

A {{<hover label="convert" line="6">}}转换{{</hover>}}转换需要 {{<hover label="convert" line="8">}}toType{{</hover>}}定义输出数据类型。

```yaml {label="convert",copy-lines="none"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.label["numberToString"]
    transforms:
      - type: convert
        convert:
          toType: string
```

支持的 `toType` 值: {{< table "table table-sm table-hover" >}}| `toType` 值 | 描述 | | -- | | -- | | `bool` | 一个 `true` 或 `false` 的布尔值。{{< /table >}}

#### 将字符串转换为布尔值

从字符串转换为 `bool` 时，crossplane 认为字符串值 `1`, `t`, `T`, `TRUE`, `True` 和 `true` 等于布尔值 `True`。

字符串 `0`、`f`、`F`、`FALSE`、`False` 和 `false` 等于布尔值 `False`。

#### 将数字转换为布尔值

Crossplane 认为整数 "1 "和浮点 "1.0 "等于布尔值 "True"。 任何其他整数或浮点值都是 "False"。

#### 将布尔值转换为数字

crossplane 将布尔值 `True` 转换为整数 `1` 或 float64 `1.0`。

false "值转换为整数 "0 "或 float64"0.0

#### 将字符串转换为 float64

字符串转换为{{<hover label="format" line="3">}}浮点64{{</hover>}}crossplane 支持可选的{{<hover label="format" line="4">}}格式: 数量{{</hover>}}字段。

被引用 {{<hover label="format" line="4">}}格式: 数量{{</hover>}}可将 "megabyte "的后缀 "M "或 "megabit "的后缀 "Mi "转换为正确的 float64 值。

{{<hint "note" >}}有关支持的后缀的完整列表，请参阅 [Go 语言文档](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/resource#Quantity)。{{</hint >}}

添加 {{<hover label="format" line="4">}}格式: 数量{{</hover>}}到{{<hover label="format" line="1">}}转换{{</hover>}}对象，以启用数量后缀支持。

```yaml {label="format",copy-lines="all"}
- type: convert
  convert:
   toType: float64
   format: quantity
```

#### 将字符串转换为对象

crossplane 可将 JSON 字符串转换为对象。

添加 {{<hover label="object" line="4">}}format: json{{</hover>}}到{{<hover label="object" line="1">}}转换{{</hover>}}对象中添加 format: json，这是本次转换唯一支持的字符串格式。

```yaml {label="object",copy-lines="all"}
- type: convert
  convert:
   toType: object
   format: json
```

{{< hint "tip" >}}这种转换对于修补对象中的键非常有用。{{< /hint >}}

下面的示例为资源添加了一个标记，标记的{{<hover label="patch-key" line="8">}}自定义键{{</hover>}}:

```yaml {label="patch-key",copy-lines="all"}
- type: FromCompositeFieldPath
      fromFieldPath: spec.clusterName
      toFieldPath: spec.forProvider.tags
      transforms:
      - type: string
        string:
          type: Format
          fmt: '{"kubernetes.io/cluster/%s": "true"}'
      - type: convert
        convert:
          toType: object
          format: json
```

#### 将字符串转换为数组

crossplane 可将 JSON 字符串转换为数组。

添加 {{<hover label="array" line="4">}}format: json{{</hover>}}到{{<hover label="array" line="1">}}转换{{</hover>}}对象中添加 format: json，这是本次转换唯一支持的字符串格式。

```yaml {label="array",copy-lines="all"}
- type: convert
  convert:
   toType: array
   format: json
```

### 地图变换

地图 {{<hover label="map" line="6">}}映射{{</hover>}}转换类型将输入值映射为输出值。

{{< hint "tip" >}}地图 {{<hover label="map" line="6">}}地图{{</hover>}}转换用于将通用的地区名称（如 "美国 "或 "欧盟"）转换为 Providers 特定的地区名称。{{< /hint >}}

地图 {{<hover label="map" line="6">}}地图{{</hover>}}转换将来自 {{<hover label="map" line="3">}}字段路径{{</hover>}}中列出的选项进行比较。 {{<hover label="map" line="6">}}中列出的选项进行比较。{{</hover>}}.

如果crossplane找到了该值，crossplane会将映射值放入 {{<hover label="map" line="4">}}到字段路径{{</hover>}}.

{{<hint "note" >}}如果找不到 Values，crossplane 会为补丁抛出错误。{{< /hint >}}

{{<hover label="map" line="3">}}spec.field1{{</hover>}}是字符串{{<hover label="map" line="8">}}"field1-text "字符串{{</hover>}}则 crossplane 会被引用字符串{{<hover label="map" line="8">}}firstField{{</hover>}}作为{{<hover label="map" line="4">}}Annotations{{</hover>}}.

如果{{<hover label="map" line="3">}}spec.field1{{</hover>}}是字符串{{<hover label="map" line="8">}}"field2-text "字符串{{</hover>}}则 crossplane 会被引用字符串{{<hover label="map" line="8">}}secondField{{</hover>}}作为{{<hover label="map" line="4">}}Annotations{{</hover>}}.

```yaml {label="map",copy-lines="none"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: map
        map:
          "field1-text": "firstField"
          "field2-text": "secondField"
```

在本例中，spec.field1 的值为{{<hover label="map" line="3">}}spec.field1{{</hover>}}的值是{{<hover label="comositeMap" line="5">}}field1-text{{</hover>}}.

```yaml {label="comositeMap",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-twx7n
Spec:
  # Removed for brevity
  field1:         field1-text
```

应用于托管资源的 Annotations 是{{<hover label="mrMap" line="4">}}firstField{{</hover>}}.

```yaml {label="mrMap",copy-lines="none"}
$ kubectl describe bucket
Name:         my-example-claim-twx7n-ndb2f
Annotations:  crossplane.io/composition-resource-name: bucket1
              myAnnotation: firstField
# Removed for brevity.
```

#### 匹配转换

比赛 {{<hover label="match" line="6">}}匹配{{</hover>}}变换与 `map` 变换类似。

比赛 {{<hover label="match" line="6">}}匹配{{</hover>}}变换增加了对正则表达式和精确字符串的支持，并能在不匹配时提供默认值。

A {{<hover label="match" line="7">}}匹配{{</hover>}}对象需要一个{{<hover label="match" line="8">}}模式{{</hover>}}对象。

模式 {{<hover label="match" line="8">}}模式{{</hover>}}是一个或多个模式的列表，用于尝试匹配输入值。

```yaml {label="match",copy-lines="1-8"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              # Removed for brevity
            - type: regexp
              # Removed for brevity
```

匹配 {{<hover label="match" line="8">}}模式{{</hover>}}可以是{{<hover label="match" line="9">}}类型: 字面{{</hover>}}匹配精确字符串或{{<hover label="match" line="11">}}类型: regexp{{</hover>}}来匹配正则表达式。

{{<hint "note" >}}crossplane 会在第一个模式匹配后停止处理匹配。{{< /hint >}}

#### 匹配精确字符串

被引用的 {{<hover label="matchLiteral" line="8">}}模式{{</hover>}}用{{<hover label="matchLiteral" line="9">}}type: literal{{</hover>}}来匹配精确字符串。

成功匹配后，Crossplane 会提供{{<hover label="matchLiteral" line="11">}}结果: {{</hover>}}到补丁 {{<hover label="matchLiteral" line="4">}}到字段路径{{</hover>}}.

```yaml {label="matchLiteral"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "field1-text"
              result: "matchedLiteral"
```

#### 匹配正则表达式

被引用的 {{<hover label="matchRegex" line="8">}}模式{{</hover>}}用{{<hover label="matchRegex" line="9">}}type: regexp{{</hover>}}来匹配正则表达式。{{<hover label="matchRegex" line="10">}}regexp{{</hover>}}键，其中包含要匹配的正则表达式的值。

成功匹配后，Crossplane 会提供{{<hover label="matchRegex" line="11">}}结果: {{</hover>}}到补丁 {{<hover label="matchRegex" line="4">}}到字段路径{{</hover>}}.

```yaml {label="matchRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: regexp
              regexp: '^field1.*'
              result: "foundField1"
```

#### 使用默认值

Provider 可以选择提供一个默认值，以便在没有匹配模式时使用。

默认值可以是原始输入值，也可以是已定义的默认值。

被引用{{<hover label="defaultValue" line="12">}}fallbackTo: Values{{</hover>}}来提供默认值。

例如，如果字符串{{<hover label="defaultValue" line="10">}}未知字符串{{</hover>}}不匹配，则 crossplane 会提供{{<hover label="defaultValue" line="12">}}Values{{</hover>}}{{<hover label="defaultValue" line="13">}}StringNotFound{{</hover>}}的{{<hover label="defaultValue" line="4">}}字段路径{{</hover>}}

```yaml {label="defaultValue"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "UnknownString"
              result: "foundField1"
          fallbackTo: Value
          fallbackValue: "StringNotFound"
```

要使用原始输入作为回退值，请使用{{<hover label="defaultInput" line="12">}}fallbackTo: 输入{{</hover>}}.

crossplane 被引用原来的{{<hover label="defaultInput" line="3">}}字段路径{{</hover>}}输入的{{<hover label="defaultInput" line="4">}}到字段路径{{</hover>}}Values 值。

```yaml {label="defaultInput"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "UnknownString"
              result: "foundField1"
          fallbackTo: Input
```

#### 数学变换

被引用 {{<hover label="math" line="6">}}数学{{</hover>}}转换对输入进行乘法运算，或应用最小值或最大值。

{{<hint "important">}}A {{<hover label="math" line="6">}}数学{{</hover>}}变换只支持整数输入。{{< /hint >}}

```yaml {label="math",copy-lines="1-7"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          ...
```

<!-- vale Google.Headings = NO -->

#### clampMin

<!-- vale Google.Headings = YES -->

类型 {{<hover label="clampMin" line="8">}}类型: clampMin{{</hover>}}被引用的最小值，如果输入大于{{<hover label="clampMin" line="8">}}类型: clampMin{{</hover>}}Values 值。

例如{{<hover label="clampMin" line="8">}}类型: clampMin{{</hover>}}要求输入大于{{<hover label="clampMin" line="9">}}20{{</hover>}}.

如果输入低于{{<hover label="clampMin" line="9">}}20{{</hover>}}，则 crossplane 会被引用{{<hover label="clampMin" line="9">}}钳位最小{{</hover>}}值作为{{<hover label="clampMin" line="4">}}到字段路径{{</hover>}}.

```yaml {label="clampMin"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: clampMin
          clampMin: 20
```

<!-- vale Google.Headings = NO -->

#### 夹钳Max

<!-- vale Google.Headings = YES -->

类型 {{<hover label="clampMax" line="8">}}类型: clampMax{{</hover>}}如果输入值大于{{<hover label="clampMax" line="8">}}类型: clampMax{{</hover>}}Values 值。

例如{{<hover label="clampMax" line="8">}}类型: clampMax{{</hover>}}要求输入小于{{<hover label="clampMax" line="9">}}5{{</hover>}}.

如果输入值高于{{<hover label="clampMax" line="9">}}5{{</hover>}}，则 crossplane 会被引用{{<hover label="clampMax" line="9">}}钳位最大{{</hover>}}值作为{{<hover label="clampMax" line="4">}}到字段路径{{</hover>}}.

```yaml {label="clampMax"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: clampMax
          clampMax: 5
```

<!-- vale Google.Headings = NO -->

#### 倍增

<!-- vale Google.Headings = YES -->

类型 {{<hover label="multiply" line="8">}}类型: 乘法{{</hover>}}将输入值乘以 {{<hover label="multiply" line="9">}}乘以{{</hover>}}Values 的值。

例如{{<hover label="multiply" line="8">}}类型: 乘法{{</hover>}}将来自 {{<hover label="multiply" line="3">}}fromFieldPath{{</hover>}}的值乘以 {{<hover label="multiply" line="9">}}2{{</hover>}}

```yaml {label="multiply"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: multiply
          multiply: 2
```

{{<hint "note" >}}乘法 {{<hover label="multiply" line="9">}}乘{{</hover>}}值只支持整数。{{< /hint >}}

### 字符串转换

字符串 {{<hover label="string" line="6">}}字符串{{</hover>}}转换应用字符串格式化或操作字符串输入。

```yaml {label="string"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["stringAnnotation"]
    transforms:
      - type: string
        string:
          type: ...
```

字符串变换支持以下{{<hover label="string" line="7">}}类型{{</hover>}}

* [转换](#string-convert)
* [格式](#string-format)
* [加入](#加入)
* [Regexp](#regular-expression-type)
* [TrimPrefix](#trim-prefix)
* [TrimSuffix](#trim-suffix)

#### 字符串转换

类型 {{<hover label="stringConvert" line="9">}}类型: 转换{{</hover>}}根据以下转换类型之一转换输入内容: 

* `ToUpper` - 将字符串改为全部大写字母。
* `ToLower` - 将字符串改为全部小写字母。
* `ToBase64` - 从输入创建新的 base64 字符串。
* `FromBase64` - 从 base64 输入创建新的文本字符串。
* `ToJson` - 将输入字符串转换为有效的 JSON 格式。
* `ToSha1` - 创建输入字符串的 SHA-1 哈希值。
* `ToSha256` - 创建输入字符串的 SHA-256 哈希值。
* ToSha512` - 创建输入字符串的 SHA-512 哈希值。
* `ToAdler32` - 创建输入字符串的 Adler32 哈希值。

```yaml {label="stringConvert"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["FIELD1-TEXT"]
    transforms:
      - type: string
        string:
          type: Convert
          convert: "ToUpper"
```

#### 字符串格式

类型 {{<hover label="typeFormat" line="9">}}类型: 格式{{</hover>}}将 [Go string formatting](https://pkg.go.dev/fmt) 应用于输入。

```yaml {label="typeFormat"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["stringAnnotation"]
    transforms:
      - type: string
        string:
          type: Format
          fmt: "the-field-%s"
```

#### 加入

类型 {{<hover label="typeJoin" line="8">}}类型: 连接{{</hover>}}使用给定的分隔符将输入数组中的所有 Values 连接成字符串。

这种变换只适用于数组输入。

```yaml {label="typeJoin"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.inputList
    toFieldPath: spec.targetJoined
    transforms:
      - type: string
        string:
          type: Join
          join:
            separator: ","
```

#### 正则表达式类型

类型 {{<hover label="typeRegex" line="8">}}类型: Regexp{{</hover>}}提取与正则表达式匹配的输入部分。

可被引用为{{<hover label="typeRegex" line="11">}}组{{</hover>}}匹配正则表达式捕获组。 默认情况下，crossplane 会匹配整个正则表达式。

```yaml {label="typeRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["euRegion"]
    transforms:
      - type: string
        string:
          type: Regexp
          regexp:
            match: '^eu-(.*)-'
            group: 1
```

#### 饰边前缀

类型 {{<hover label="typeRegex" line="8">}}类型: TrimPrefix{{</hover>}}移除匹配字符串和前面的所有字符。

```yaml {label="typeRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["north-1"]
    transforms:
      - type: string
        string:
          type: TrimPrefix
          trim: `eu-
```

#### 后缀

类型 {{<hover label="typeRegex" line="8">}}类型: TrimSuffix{{</hover>}}移除匹配字符串和所有后续字符。

```yaml {label="typeRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    transforms:
      - type: string
        string:
          type: TrimSuffix
          trim: `-north-1'
```

## 补丁政策

crossplane 支持两种类型的修补策略: 

* 从字段路径
* 合并选项

<!-- vale Google.Headings = NO -->

### fromFieldPath 策略

<!-- vale Google.Headings = YES -->

在补丁上引用 `fromFieldPath: Required` 策略要求 `fromFieldPath` 必须存在于 Composition 资源中。

{{<hint "tip" >}}如果资源补丁不起作用，应用 `fromFieldPath: Required` 策略可能会在 Composition 资源中产生错误，从而帮助排除故障。{{< /hint >}}

默认情况下，crossplane 应用 "fromFieldPath: Optional "策略。 如果 "fromFieldPath: Optional "不存在，crossplane 会忽略补丁。

有{{<hover label="required" line="6">}}fromFieldPath: Required{{</hover>}}的情况下，复合资源会产生错误。{{<hover label="required" line="6">}}fromFieldPath{{</hover>}}不存在，则复合资源会产生错误。

```yaml {label="required"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    policy:
      fromFieldPath: Required
```

### 合并选项

默认情况下，应用补丁时目标数据会被覆盖。 使用{{<hover label="merge" line="6">}}合并选项{{</hover>}}允许补丁在不覆盖数组和对象的情况下合并它们。

被引用数组输入时，使用{{<hover label="merge" line="7">}}appendSlice: true{{</hover>}}将数组数据追加到现有数组的末尾。

被引用对象时，请使用{{<hover label="merge" line="8">}}keepMapValues: true{{</hover>}}来保留现有对象键值。 补丁会更新输入数据和目标数据之间的任何匹配键值。

```yaml {label="merge"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    policy:
      mergeOptions:
        appendSlice: true
        keepMapValues: true
```
