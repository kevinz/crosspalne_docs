---
title: Compositions
weight: 30
aliases:
  - composition
description: "Compositions are a template for creating Crossplane resources"
---

Composition 是将多个受管资源创建为单一对象的模板。

Composition 将单个受管资源组合成一个更大的、可重复使用的解决方案。

Composition 模板将所有这些单独的资源连接在一起。

{{<expand "Confused about Compositions, XRDs, XRs and Claims?" >}}crossplane 有四个核心组件，用户通常会把它们混为一谈: 

* Composition - 本页。定义如何创建资源的模板。
* [Composition Resource Definition]({{<ref "./composite-resource-definitions">}}) (`XRD`) - 一种自定义 API 规范。
* [复合资源]({{<ref "./composite-resources">}}) (`XR`) - 通过使用 Composition Resource Definition 中定义的自定义 API 创建。XRs 使用 Composition 模板来创建新的托管资源。
* [索赔]({{<ref "./claims" >}}) (`XRC`) - 类似于 Composition Resource，但具有名称空间范围。

{{</expand >}}

## 创建 Composition

创作 Composition 包含以下内容: 

* [资源模板](#resource-templates) 定义要创建的资源。
* [启用复合资源](#enabling-composite-resources) 以引用此 Composition 模板。

<!-- vale Google.WordList = NO -->

Composition 还支持可选功能: 

* [修改和修补](#changing-resource-fields) 资源设置。
* [存储连接详情](#storing-connection-details) 和由托管资源生成的秘密。
* 使用[Composition Functions](#use-composition-functions)，利用自定义程序将资源模板化。
* 创建一个[资源就绪时的自定义检查](#resource-readiness-checks)来使用。

<!-- vale Google.WordList = YES -->

### 资源模板

资源{{<hover label="resources" line="4">}}资源{{</hover>}}字段的{{<hover label="resources" line="3">}}规格{{</hover>}}的资源字段定义了 Composition 资源创建的内容集。

{{<hint "note" >}}有关复合资源的更多信息，请参阅[复合资源页面]({{<ref "./composite-resources" >}}).{{< /hint >}}

例如，Composition 可以定义一个模板来同时创建虚拟机和相关的存储桶。

资源 {{<hover label="resources" line="4">}}资源{{</hover>}}字段列出了带有{{<hover label="resources" line="5">}}名称。{{</hover>}}该名称用于识别 Composition 中的资源，与 Providers 被引用的外部名称无关。

#### 模板一个受管资源

的内容{{<hover label="resources" line="6">}}基{{</hover>}}的内容与创建独立的[managed resource]({{<ref "./managed-resources">}}).

本例被引用[upbound 的 Provider AWS](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.35.0)来定义一个 S3 存储 {{<hover label="resources" line="8">}}存储桶{{</hover>}}和 EC2 计算 {{<hover label="resources" line="15">}}实例{{</hover>}}.

在定义了{{<hover label="resources" line="7">}}和{{</hover>}}和{{<hover label="resources" line="8">}}类型{{</hover>}}后，定义{{<hover label="resources" line="10">}}spec.forProvider{{</hover>}}字段，定义资源的设置。

```yaml {copy-lines="none",label="resources"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: StorageBucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: "us-east-2"
    - name: VM
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            ami: ami-0d9858aa3c6322f73
            instanceType: t2.micro
            region: "us-east-2"
```

当[复合资源]({{<ref "./composite-resources" >}})使用此 Composition 模板时，复合资源会创建两个新的托管资源，其中包含所有提供的{{<hover label="resources" line="17">}}spec.forProvider{{</hover>}}设置创建两个新的托管资源。

规格 {{<hover label="resources" line="16">}}规范{{</hover>}}支持在托管资源中使用的任何设置，包括应用 "Annotations "和 "labels "或使用特定的 "providerConfigRef"。

{{<hint "note" >}}Composition 允许将 "metadata.name "应用到资源的{{<hover label="resources" line="16">}}但会忽略它。{{</hover>}}metadata.name "字段不会影响 crossplane 中的受管资源名称或 Provider 中的外部资源名称。

在资源上使用 `crossplane.io/external-name` 注解来影响外部资源名称。{{< /hint >}}

#### 一个 ProviderConfig 模板

Composition 可以像定义托管资源一样定义 ProviderConfig。 生成 ProviderConfig 可能有助于为每个部署提供唯一的凭据。

```yaml {copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: my-aws-provider-config
      base:
        apiVersion: aws.upbound.io/v1beta1
        kind: ProviderConfig
        spec:
          source: Secret
          secretRef:
            namespace: crossplane-system
            name: aws-secret
            key: creds
```

#### 另一个 Composition 资源模板

Composition 可以被引用其他 Composite 资源来定义更复杂的模板。

常见的用例是一个 Composition 使用其他 Composition。 例如，创建一个 Composition 来创建一套标准网络资源，供其他 Composition 引用。

{{< hint "note" >}}两种 Composition 都必须有相应的 XRD。{{< /hint >}}

此示例网络 Composition 定义了创建新 AWS 虚拟网络所需的资源集。 其中包括一个{{<hover label="xnetwork" line="8">}}VPC{{</hover>}},{{<hover label="xnetwork" line="13">}}互联网网关{{</hover>}}和{{<hover label="xnetwork" line="18">}}子网{{</hover>}}.

```yaml {copy-lines="none",label="xnetwork"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: vpc-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        # Removed for Brevity
    - name: gateway-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        # Removed for Brevity
    - name: subnet-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Subnet
        # Removed for Brevity
  compositeTypeRef:
    apiVersion: aws.platformref.upbound.io/v1alpha1
    kind: XNetwork
```

复合类型 {{<hover label="xnetwork" line="20">}}复合类型{{</hover>}}为该 Composition 提供了一个{{<hover label="xnetwork" line="21">}}apiVersion{{</hover>}}和{{<hover label="xnetwork" line="22">}}类型{{</hover>}}以便在另一个 Composition 中引用。

{{<hint "note" >}}启用 Composition 资源](#enabling-composite-resources) 部分描述了{{<hover label="xnetwork" line="20">}}compositeTypeRef{{</hover>}}字段。{{< /hint >}}

第二个 Composition 定义了其他资源，在本例中是 AWS Elastic Kubernetes 集群，可以引用之前的{{<hover label="xnetwork" line="22">}}XNetwork{{</hover>}}

```yaml {copy-lines="none",label="xcluster"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: nested-network-composition
      base:
        apiVersion: aws.platformref.upbound.io/v1alpha1
        kind: XNetwork
        # Removed for brevity
    - name: eks-cluster-resource
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: Cluster
        # Removed for brevity
```

当复合资源从该 Composition 中创建所有受管资源时，由{{<hover label="xcluster" line="8">}}XNetwork{{</hover>}}定义的资源与 EKS {{<hover label="xcluster" line="13">}}集群{{</hover >}}.

{{<hint "note" >}}这个缩写示例来自 upbound [AWS 参考平台](https://github.com/upbound/platform-ref-aws)。

在参考平台的 [package 目录](https://github.com/upbound/platform-ref-aws/blob/main/apis/cluster/composition.yaml) 中查看完整的 Composition。{{</hint >}}

#### 跨资源参考

在 Composition 内部，一些资源会使用其他资源的标识符或名称。 例如，创建一个新的 `network` 并将网络标识符应用于虚拟机。

Composition 中的资源可以通过匹配标签或_controller reference_（控制器引用）来交叉引用其他资源。

{{<hint "note" >}}Providers 允许按资源匹配标签和控制器引用。 请查看特定 Provider 资源的文档，了解支持哪些功能。

不同 Provider 不支持匹配标签和控制器。{{< /hint >}}

##### 匹配资源标签

要匹配资源标签，首先应用{{<hover label="matchlabel" line="11">}}标签{{</hover>}}并使用{{<hover label="matchlabel" line="19">}}匹配标签{{</hover>}}在第二个资源中使用

此示例创建了一个 AWS{{<hover label="matchlabel" line="7">}}角色{{</hover>}}并应用{{<hover label="matchlabel" line="11">}}标签。{{</hover>}}第二个资源是一个 {{<hover label="matchlabel" line="14">}}角色策略附件{{</hover>}}需要附加到现有的 "角色"。

被引用资源的{{<hover label="matchlabel" line="19">}}角色选择器的{{</hover>}}可确保该{{<hover label="matchlabel" line="14">}}角色策略附件{{</hover>}}引用匹配的{{<hover label="matchlabel" line="7">}}角色{{</hover>}}，即使不知道唯一的角色标识符。

```yaml {label="matchlabel",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: Role
        name: iamRole
        metadata:
          labels:
            role: controlplane
    - base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        name: iamPolicy
        spec:
          forProvider:
            roleSelector:
              matchLabels:
                role: controlplane
  # Removed for brevity
```

##### 匹配控制器参考

匹配控制器引用可确保匹配的资源在同一 Composition 资源中。

只匹配控制器参考信息可简化匹配过程，无需标签或更多信息。

例如，创建一个 AWS{{<hover label="controller1" line="14">}}互联网网关{{</hover>}}需要一个{{<hover label="controller1" line="7">}}VPC{{</hover>}}.

互联网网关 {{<hover label="controller1" line="14">}}网关{{</hover>}}可以匹配一个标签，但该 Composition 创建的每个 VPC 都共享同一个标签。

被引用 {{<hover label="controller1" line="19">}}matchControllerRef{{</hover>}}只能匹配在创建了{{<hover label="controller1" line="14">}}互联网网关{{</hover>}}.

```yaml {label="controller1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-vpc
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        name: my-gateway
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
# Removed for brevity
```

资源可以同时匹配标签和控制器引用，以匹配更大的 Composition 资源中的特定资源。

例如，此 Composition 会创建两个{{<hover label="controller2" line="17">}}VPC{{</hover>}}资源，但{{<hover label="controller2" line="27">}}网关{{</hover>}}必须只匹配一个。

应用 {{<hover label="controller2" line="21">}}标签{{</hover>}}对第二个 {{<hover label="controller2" line="17">}}VPC{{</hover>}}允许{{<hover label="controller2" line="27">}}互联网网关{{</hover>}}匹配标签{{<hover label="controller2" line="34">}}类型: 互联网{{</hover>}}并只匹配同一复合资源中的对象，并使用{{<hover label="controller2" line="32">}}matchControllerRef{{</hover>}}.

```yaml {label="controller2",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-first-vpc
        metadata:
          labels:
            type: backend
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-second-vpc
        metadata:
          labels:
            type: internet
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        name: my-gateway
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
              matchLabels:
                type: internet
# Removed for brevity
```

### 启用 Composition 资源

Composition 只是一个定义如何创建托管资源的模板。 Composition 限制哪些复合资源可以使用此模板。

一个 Composition 的 {{<hover label="typeref" line="6">}}的{{</hover>}}定义了哪种复合资源类型可以使用此 Composition。

{{<hint "note" >}}有关复合资源的更多信息，请参阅[复合资源页面]({{<ref "./composite-resources" >}}).{{< /hint >}}

在 Composition 的{{<hover label="typeref" line="5">}}规范中{{</hover>}}定义 Composition 资源{{<hover label="typeref" line="7">}}apiVersion{{</hover>}}和{{<hover label="typeref" line="8">}}类型{{</hover>}}被引用。

```yaml {label="typeref",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: dynamodb-with-bucket
spec:
  compositeTypeRef:
    apiVersion: custom-api.example.org/v1alpha1
    kind: database
  # Removed for brevity
```

### 更改资源字段

大多数 Composition 都需要自定义资源字段，包括应用唯一密码、修改资源部署位置或应用标签或 Annotations。

更改资源的主要方法是使用资源[修补和转换]({{<ref "./patch-and-transform" >}}补丁和变换可以匹配特定的输入字段，对其进行修改并应用到被管理的资源中。

{{<hint "important" >}}关于创建补丁和变换及其选项的详细信息，请参阅[补丁和变换页面]({{<ref "./patch-and-transform" >}}).

本节介绍如何将修补程序和变换应用到 Composition 中。{{< /hint >}}

使用{{<hover label="patch" line="13">}}补丁{{</hover>}}字段应用补丁。

例如，将{{<hover label="patchClaim" line="6">}}位置{{</hover>}}claims 中提供的位置，并将其应用到{{<hover label="patch" line="12">}}区域{{</hover>}}值。

```yaml {copy-lines="none",label="patchClaim"}
apiVersion: example.org/v1alpha1
kind: ExampleClaim
metadata:
  name: my-example-claim
spec:
  location: "eu-north-1"
```

Composition 补丁被引用为{{<hover label="patch" line="15">}}字段路径{{</hover>}}来匹配{{<hover label="patchClaim" line="6">}}位置{{</hover>}}字段，而{{<hover label="patch" line="16">}}字段路径{{</hover>}}来定义要在 Composition 中更改的字段。

```yaml {copy-lines="none",label="patch"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
    - name: s3Bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: "us-east-2"
      patches:
      - type: FromCompositeFieldPath
        fromFieldPath: "spec.location"
        toFieldPath: "spec.forProvider.region"
```

#### 补丁套件

有些 Composition 的资源需要应用相同的补丁，与其重复相同的 "patches "字段，资源可以引用单一的 "patchSet"。

定义一个 {{<hover label="patchset" line="5">}}补丁集{{</hover>}}的{{<hover label="patchset" line="6">}}名称{{</hover>}}和{{<hover label="patchset" line="7">}}补丁{{</hover>}}操作。

然后 apply{{<hover label="patchset" line="5">}}补丁集{{</hover>}}应用到每个资源{{<hover label="patchset" line="16">}}类型: patchSet{{< /hover >}}，引用 {{<hover label="patchset" line="6">}}名称{{< /hover >}}中的{{<hover label="patchset" line="17">}}补丁集名称{{< /hover >}}字段中的名称。

```yaml {copy-lines="none",label="patchset"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  patchSets:
    - name: reusable-patch
      patches:
      - type: FromCompositeFieldPath
        fromFieldPath: "location"
        toFieldPath: "spec.forProvider.region"
  resources:
    - name: first-resource
      base:
      # Removed for Brevity
      patches:
        - type: PatchSet
          patchSetName: reusable-patch
    - name: second-resource
      base:
      # Removed for Brevity
      patches:
        - type: PatchSet
          patchSetName: reusable-patch
```

#### 补丁与 EnvironmentConfigs

Crossplane 使用 EnvironmentConfigs 来创建内存数据存储。 Composition 可以从该数据存储中读写，这是补丁程序的一部分。

{{<hint "important" >}}EnvironmentConfigs 是一项 alpha 功能，默认情况下不会启用。{{< /hint >}}

EnvironmentConfigs 可以预定义 Composition 可以引用的数据，或者 Composite Resource 可以将数据写入其内存环境，供其他资源读取。

<!-- vale off -->

{{< hint "note" >}}

<!-- vale on -->

阅读 [EnvironmentConfigs]({{<ref "./environment-configs" >}}) 页面，了解有关使用 EnvironmentConfigs 的更多信息。{{< /hint >}}

要使用 EnvironmentConfigs 引用补丁，首先要使用{{<hover label="envselect" line="6">}}environment.environmentConfigs{{</hover>}}.

<!-- vale Google.Quotes = NO -->

<!-- vale gitlab.SentenceLength = NO -->

<!-- ignore false positive -->

使用[引用]({{<ref "./managed-resources#matching-by-name-reference" >}}) 或 [selector]({{<ref "./managed-resources#matching-by-selector" >}}) 来标识要使用的 EnvironmentConfigs。

<!-- vale Google.Quotes = YES -->

```yaml {label="envselect",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
    environmentConfigs:
      - ref:
          name: example-environment
  resources:
  # Removed for Brevity
```

<!-- these two sections are duplicated in the environment-configs doc -->

##### Patch 一个 Composition 资源

要在 Composition 资源和内存环境之间打补丁，请使用{{< hover label="xrpatch" line="7">}}补丁{{</hover>}}内的{{< hover label="xrpatch" line="5">}}环境{{</hover>}}.

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

单个资源可以使用写入其内存环境的任何数据。

##### 补丁单个资源

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

EnvironmentConfigs]({{<ref "./environment-configs" >}}) 页面有更多关于 EnvironmentConfigs 选项和用法的信息。

#### 被引用的组成函数

Composition 函数（简称函数）是模板化 crossplane 资源的自定义程序。 您可以使用 Go 或 Python 等通用编程语言编写函数模板化资源。 使用通用编程语言，函数可以使用更高级的逻辑模板化资源，如循环和条件。

{{<hint "important" >}}合成功能是一项测试版功能。 Crossplane 默认启用测试版功能。 在[合成功能]({{<ref "./composition-functions#disable-composition-functions">}}) 页面解释了如何禁用 Composition 功能。{{< /hint >}}

要使用合成功能，请设置合成{{<hover label="xfn" line="6">}}模式{{</hover>}}为{{<hover label="xfn" line="6">}}Pipelines{{</hover>}}.

定义一个 {{<hover label="xfn" line="7">}}Pipelines{{</hover>}}的{{<hover label="xfn" line="8">}}步骤。{{</hover>}}每个{{<hover label="xfn" line="8">}}步骤{{</hover>}}都会调用一个函数。

每个 {{<hover label="xfn" line="8">}}步骤{{</hover>}}被引用一个{{<hover label="xfn" line="9">}}函数{{</hover>}}来引用{{<hover label="xfn" line="10">}}名称{{</hover>}}的名称。

某些函数还允许您指定一个{{<hover label="xfn" line="11">}}输入。{{</hover>}}函数定义了{{<hover label="xfn" line="13">}}输入{{</hover>}}输入。

{{<hint "important" >}}被引用的构成 {{<hover label="xfn" line="6">}}模式: Pipelines{{</hover>}}不能使用 `resources` 字段指定资源模板。

使用功能 "修补和转换 "创建资源模板。{{< /hint >}}

本示例使用了函数 Pipelines 和 Transform。 函数 Pipelines 和 Transform 是一个实现 crossplane 资源模板的函数。 您可以使用函数 Pipelines 和 Transform 与其他函数一起在管道中指定资源模板。

```yaml {label="xfn",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  # Removed for Brevity
  mode: Pipeline
  pipeline:
  - step: patch-and-transform
    functionRef:
      name: function-patch-and-transform
    input:
      apiVersion: pt.fn.crossplane.io/v1beta1
      kind: Resources
      resources:
      - name: storage-bucket
        base:
          apiVersion: s3.aws.upbound.io/v1beta1
          kind: Bucket
          spec:
            forProvider:
              region: "us-east-2"
```

请阅读 [Composition functions]({{<ref "./composition-functions">}}) 页面，了解构建和使用 Composition 函数的详情。

### 存储连接详情

一些托管资源会生成独特的详细信息，如用户名、密码、IP 地址、端口或其他连接详细信息。

当 Composition 中的资源创建连接详情时，crossplane 会为每个生成连接详情的托管资源创建一个 Kubernetes secret 对象。

{{<hint "note">}}本节讨论创建 Kubernetes 秘密。 crossplane 还支持使用外部秘密存储，如 [HashiCorp Vault](https://www.vaultproject.io/)。

请阅读[外部秘密存储指南]({{<ref "/knowledge-base/integrations/vault-as-secret-store">}}) 以获取更多有关与外部秘密存储一起使用 crossplane 的信息。{{</hint >}}

#### 复合资源组合秘密

crossplane 可以将 Composition 内部资源生成的所有 secret 合并为一个 Kubernetes secret，并可选择复制该 secret 对象用于 [claims]({{<ref "./claims#claim-connection-secrets">}}).

设置{{<hover label="writeConn" line="5">}}的值设置为{{</hover>}}的值设置为 crossplane 应存储组合秘密对象的名称空间。

```yaml {copy-lines="none",label="writeConn"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  writeConnectionSecretsToNamespace: my-namespace
  resources:
  # Removed for brevity
```

#### 汇编资源秘密

在{{<hover label="writeConnRes" line="10">}}规格{{</hover>}}中，定义{{<hover label="writeConnRes" line="13">}}writeConnectionSecretToRef{{</hover>}}的{{<hover label="writeConnRes" line="14">}}namespace{{</hover>}}和{{<hover label="writeConnRes" line="15">}}名称{{</hover>}}资源的 secret 对象。

如果{{<hover label="writeConnRes" line="13">}}writeConnectionSecretToRef{{</hover>}}未定义，则 crossplane 不会向 secret 写入任何密钥。

```yaml {label="writeConnRes"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
```

crossplane 会保存一个秘密，其名称为{{<hover label="viewComposedSec" line="3">}}名称{{</hover>}}的秘密保存在{{<hover label="writeConnRes" line="14">}}namespace{{</hover>}}Providers.

```shell {label="viewComposedSec"}
kubectl get secrets -n docs
NAME TYPE DATA AGE
key1 connection.crossplane.io/v1alpha1 4 4m30s
```

{{<hint "tip" >}}

crossplane 建议使用 [Patch]({{<ref "./patch-and-transform">}}) 为每个秘密创建一个唯一的名称。

例如{{<hover label="tipPatch" line="15">}}补丁{{</hover>}}将资源的唯一标识符添加为键名。

```yaml {label="tipPatch",copy-lines="14-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  # Removed for brevity
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
        # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret"
```

{{< /hint >}}

#### 定义秘密密钥

Composition 必须用{{<hover label="conDeet" line="14">}}连接详情{{</hover>}}对象创建的特定秘钥。

{{<table "table table-sm" >}}| Secret Type | Description | | --- | --- | | | | | {{<hover label="conDeet" line="16">}}fromConnectionSecretKey{{</hover>}}| 创建与资源生成的 Secret 密钥相匹配的 Secret 密钥。 {{<hover label="conDeet" line="18">}}fromFieldPath{{</hover>}}| 创建与资源的字段路径相匹配的秘钥。 {{<hover label="conDeet" line="20">}}Values{{</hover>}}| 创建带有预定义值的密钥。{{< /table >}}

{{<hint "note">}}值 {{<hover label="conDeet" line="20">}}Values{{</hover>}}类型必须被引用为字符串值。

值 {{<hover label="conDeet" line="20">}}Values{{</hover>}}不会添加到单个资源 secret 对象中。{{<hover label="conDeet" line="20">}}Values{{</hover>}}只出现在组合的 Composition 资源秘密中。{{< /hint >}}

```yaml {label="conDeet",copy-lines="none"}
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        # Removed for brevity
        spec:
          forProvider:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - name: myUsername
          fromConnectionSecretKey: username
        - name: myFieldSecret
          fromFieldPath: spec.forProvider.user
        - name: myStaticSecret
          value: "docs.crossplane.io"
```

连接{{<hover label="conDeet" line="14">}}连接详情{{</hover>}}可以引用资源中的 secret，并使用{{<hover label="conDeet" line="16">}}从连接秘钥{{</hover>}}或资源中另一个字段的{{<hover label="conDeet" line="18">}}字段路径{{</hover>}}或静态定义的值。{{<hover label="conDeet" line="20">}}Values{{</hover>}}.

crossplane 将秘钥设置为{{<hover label="conDeet" line="15">}}名称{{</hover>}}Values 值。

描述秘密，查看秘密对象内的密钥。

{{<hint "tip" >}}如果有多个资源生成具有相同密钥名称的秘密，则 crossplane 只保存一个值。

被引用的自定义 {{<hover label="conDeet" line="15">}}名称{{</hover>}}创建唯一的 Secret 密钥。{{< /hint >}}

{{<hint "important">}}crossplane 只添加在{{<hover label="conDeet" line="16">}}连接详情{{</hover>}}中定义的任何连接秘密。{{<hover label="conDeet" line="16">}}连接详情{{</hover>}}都不会添加到组合秘密对象中。{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret
Name:         my-access-key-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
myUsername:      20 bytes
myFieldSecret:   24 bytes
myStaticSecret:  18 bytes
```

{{<hint "note" >}}CompositeResourceDefinition 还可以限制 Crossplane 从组合资源中存储哪些密钥。 默认情况下，XRD 会将组合资源 "connectionDetails "中列出的所有密钥写入组合密钥对象。

请阅读 [CompositeResourceDefinition 文档]({{<ref "composite-resource-definitions#manage-connection-secrets">}}) 获取更多关于限制秘钥的信息。{{< /hint >}}

有关连接秘密的更多信息，请阅读[连接秘密知识库文章]({{<ref "/knowledge-base/guides/connection-details">}}).

{{<hint "warning">}}您不能更改{{<hover label="conDeet" line="16">}}连接详情{{</hover>}}您必须删除并重新创建 Composition 才能更改{{<hover label="conDeet" line="16">}}连接细节{{</hover>}}.{{</hint >}}

#### 将连接详细信息存储到外部秘密存储中

crossplane [外部秘密存储]({{<ref "/knowledge-base/integrations/vault-as-secret-store" >}}) 将秘密和连接详情写入外部秘密存储，如 HashiCorp Vault。

{{<hint "important" >}}外部秘密存储是 alpha 功能。

crossplane 默认禁用外部秘密存储。{{< /hint >}}

被引用{{<hover label="gcp-storeconfig"
line="11">}}publishConnectionDetailsWithStoreConfigRef{{</hover>}}代替 `writeConnectionSecretsToNamespace` 来定义{{<hover label="gcp-storeconfig" line="2">}}存储配置{{</hover>}}以保存连接详细信息。

例如，被引用的{{<hover label="gcp-storeconfig" line="2">}}存储配置{{</hover>}}的{{<hover label="gcp-storeconfig" line="4">}}名称为{{</hover>}}"Vault "的存储配置，使用{{<hover label="gcp-storeconfig" line="12">}}publishConnectionDetailsWithStoreConfigRef.name{{</hover>}}与 {{<hover label="gcp-storeconfig" line="4">}}名称相匹配的{{</hover>}}在本例中为 "vault"。

```yaml {label="gcp-storeconfig",copy-lines="none"}
apiVersion: gcp.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
# Removed for brevity.
---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  publishConnectionDetailsWithStoreConfigRef: 
    name: vault
  resources:
  # Removed for brevity
```

更多详情请阅读 [External Secret Stores]({{<ref "/knowledge-base/integrations/vault-as-secret-store" >}}) 集成指南。

### 资源准备情况检查

默认情况下，当所有已创建资源的状态均为 "类型: 就绪 "和 "状态: 真 "时，Crossplane 会将 Composition 资源或 claims 视为 "已就绪"。

有些资源（例如 ProviderConfig）没有 Kubernetes 状态，永远不会被视为 "就绪"。

自定义就绪检查允许 Composition 定义资源 "就绪 "需要满足的自定义条件。

{{< hint "tip" >}}如果资源必须满足多个条件才能成为 "就绪"，则使用多重就绪检查。{{< /hint >}}

<!-- vale Google.WordList = NO -->

使用{{<hover label="check" line="10" >}}准备检查{{</hover>}}字段定义自定义就绪检查。

<!-- vale Google.WordList = YES -->

检查具有{{<hover label="check" line="11" >}}类型{{</hover>}}类型定义如何匹配资源，而 {{<hover label="check" line="12" >}}字段路径{{</hover>}}的字段路径。

```yaml {label="check",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: <match type>
          fieldPath: <resource field>
```

Composition 通过以下方式支持资源字段的匹配: 

* [字符串匹配]（#match-a-string）
* [整数匹配]（#match-an-integer）
* [非空匹配](#match-that-a-field-exists)
* [始终准备就绪](#always-consider-a-resource-ready)
* [条件匹配](#match-a-condition)
* [布尔匹配](#match-a-boolean)

#### 匹配字符串

{{<hover label="matchstring" line="11">}}匹配字符串{{</hover>}}认为，当组成资源的字段值与指定字符串匹配时，该资源已准备就绪。

{{<hint "note" >}}

<!-- vale Google.WordList = NO -->

crossplane 只支持精确的字符串匹配，不支持子字符串和正则表达式的就绪检查。

<!-- vale Google.WordList = YES -->

{{</hint >}}

例如，匹配字符串{{<hover label="matchstring" line="13">}}在线{{</hover>}}字符串匹配资源的{{<hover label="matchstring" line="12">}}status.atProvider.state{{</hover>}}字段中的字符串。

```yaml {label="matchstring",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchString
          fieldPath: status.atProvider.state
          matchString: "Online"
```

#### 匹配一个整数

{{<hover label="matchint" line="11">}}匹配整数{{</hover>}}当组成资源的字段值与指定整数相匹配时，该资源将被视为已准备就绪。

{{<hint "note" >}}

<!-- vale Google.WordList = NO -->

crossplane 不支持匹配 `0`。

<!-- vale Google.WordList = YES -->

{{</hint >}}

例如，匹配数字{{<hover label="matchint" line="13">}}4{{</hover>}}资源的{{<hover label="matchint" line="12">}}status.atProvider.state{{</hover>}}字段中的数字 4。

```yaml {label="matchint",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchInteger
          fieldPath: status.atProvider.state
          matchInteger: 4
```

#### 匹配一个字段是否存在

{{<hover label="NonEmpty" line="11">}}非空{{</hover>}}认为当一个字段存在值时，组成的资源已就绪。

{{<hint "note" >}}

<!-- vale Google.WordList = NO -->

Crossplane 将数值为 "0 "或空字符串视为空。{{</hint >}}

例如，要检查资源的{{<hover label="NonEmpty" line="12">}}status.atProvider.state{{</hover>}}字段不是空的。

<!-- vale Google.WordList = YES -->

```yaml {label="NonEmpty",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: NonEmpty
          fieldPath: status.atProvider.state
```

{{<hint "tip" >}}检查 {{<hover label="NonEmpty" line="11">}}非空{{</hover>}}不需要设置任何其他字段。{{< /hint >}}

#### 始终考虑准备好资源

{{<hover label="none" line="11">}}无{{</hover>}}认为组成的资源在创建后立即就绪。 在声明资源就绪之前，crossplane 不会等待任何其他条件。

例如，请考虑{{<hover label="none" line="7">}}我的资源{{</hover>}}一创建就准备就绪。

```yaml {label="none",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: None
```

#### 匹配条件

{{<hover label="condition" line="11">}}条件{{</hover>}}认为组成的资源在找到预期的条件类型时已准备就绪，其 `status.conditions` 中有预期的状态。

例如，请考虑{{<hover label="condition" line="7">}}我的资源{{</hover>}}，如果有一个类型为{{<hover label="condition" line="13">}}类型的{{</hover>}}状态为{{<hover label="condition" line="14">}}成功{{</hover>}}.

```yaml {label="condition",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchCondition
          matchCondition:
            type: MyType
            status: Success
```

#### 匹配布尔值

有两种类型的检查用于匹配布尔字段: 

* `MatchTrue
* 匹配错误

MatchTrue "认为，当组成的资源中某个字段的值为 "true "时，该资源已准备就绪。

`MatchFalse` 认为，当组成的资源中某个字段的值为 `false`时，该资源已准备就绪。

例如，请考虑{{<hover label="matchTrue" line="7">}}我的资源{{</hover>}}如果{{<hover label="matchTrue" line="12">}}status.atProvider.config.status.ready{{</hover>}}为 {{<hover label="matchTrue" line="11">}}为{{</hover>}}.

```yaml {label="matchTrue",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchTrue
          fieldPath: status.atProvider.manifest.status.ready
```

{{<hint "tip" >}}检查 {{<hover label="matchTrue" line="11">}}匹配为真{{</hover>}}不需要设置任何其他字段。{{< /hint >}}

`MatchFalse` 会匹配表达值为 `false` 的准备就绪的字段。

例如，请考虑{{<hover label="matchFalse" line="7">}}我的资源{{</hover>}}如果{{<hover label="matchFalse" line="12">}}status.atProvider.config.manifest.status.pending{{</hover>}}为 {{<hover label="matchFalse" line="11">}}为 false{{</hover>}}.

```yaml {label="matchFalse",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchFalse
          fieldPath: status.atProvider.manifest.status.pending
```

{{<hint "tip" >}}检查 {{<hover label="matchFalse" line="11">}}MatchFalse{{</hover>}}不需要设置任何其他字段。{{< /hint >}}

## 验证一个构成

使用 `kubectl get composition` 查看所有可用的 Composition。

```shell {copy-lines="1"}
kubectl get composition
NAME XR-KIND XR-APIVERSION AGE
xapps.aws.platformref.upbound.io XApp aws.platformref.upbound.io/v1alpha1 123m
xclusters.aws.platformref.upbound.io XCluster aws.platformref.upbound.io/v1alpha1 123m
xeks.aws.platformref.upbound.io XEKS aws.platformref.upbound.io/v1alpha1 123m
xnetworks.aws.platformref.upbound.io XNetwork aws.platformref.upbound.io/v1alpha1 123m
xservices.aws.platformref.upbound.io XServices aws.platformref.upbound.io/v1alpha1 123m
xsqlinstances.aws.platformref.upbound.io XSQLInstance aws.platformref.upbound.io/v1alpha1 123m
```

XR-KIND` 列出了允许使用 Composition 模板的复合资源 "类型"。 XR-APIVERSION` 列出了允许使用 Composition 模板的复合资源 API 版本。

{{<hint "note" >}}kubectl get composition` 的 Output 与 `kubectl get composite` 不同。

`kubectl get composition` 列出所有可用的 Composition。

`kubectl get composite` 列出所有已创建的 Composite 资源及其相关 Composition。{{< /hint >}}

## Composition validation

在创建 Composition 时，crossplane 会自动验证其完整性，例如检查 Composition 是否成型良好: 

如果被引用 `mode: Resources`: 

* 资源 "字段不是空的。
* 所有资源要么被引用，要么没有。Composition 不能同时使用已命名和未命名的资源。
* 资源名称不能重复。
* 补丁集必须有名称。
* 需要提供 `fromFieldPath` 值的补丁必须提供。
* 需要提供 `toFieldPath` 值的补丁。
* 需要 `combine` 字段的补丁必须提供。
* 使用 `matchString` 的就绪检查不是空的。
* 使用 `matchInteger` 的就绪检查不是 `0`。
* 需要 `fieldPath` 值的就绪检查提供该值。

如果被引用 `mode: Pipeline` (Composition Functions): 

* Pipelines" 字段不是空的。
* 没有重复的步骤名称。

### 構成模式感知驗證

crossplane 还会对 Composition 执行模式感知验证。 模式验证会根据资源模式检查 "补丁"、"就绪检查 "和 "连接细节 "是否有效。 例如，根据源和目标资源模式检查补丁的源和目标字段是否有效。

{{<hint "note" >}}Composition 模式感知验证是一项测试功能。 Crossplane 默认启用测试功能。

在 crossplane pod 上设置`--enable-composition-webhook-schema-validation=false`标志，禁用模式感知验证。

Crossplane Pods]({{<ref "./pods#edit-the-deployment">}}) 页面有更多关于启用 crossplane flag 的信息。{{< /hint >}}

#### 模式感知验证模式

如果出现完整性错误，crossplane 始终会拒绝 Composition。

设置模式感知验证模式，以配置 crossplane 如何处理资源模式缺失和模式感知验证错误。

{{<hint "note" >}}如果缺少资源模式，crossplane 会跳过模式识别验证，但仍会对完整性错误返回错误，对缺少的模式返回警告或错误。{{< /hint >}}

可使用以下模式

{{< table "table table-sm table-striped" >}}| 模式 | 缺少模式 | 模式感知错误 | 完整性错误 | | -------- | -------------- |--------------------|-----------------| | `warn` | Warning | Warning | Error | | `loose` | Warning | Error | Error | | `strict` | Error | Error | Error{{< /table >}}

使用{{<hover label="mode" line="5">}}来更改 Composition 的验证模式。{{</hover>}}Annotations 来更改组合体的验证模式。

如果未指定，默认模式为 "警告"。

例如，要启用 "宽松 "模式检查，请将 Annotations 值设置为{{<hover label="mode" line="5">}}松{{</hover>}}.

```yaml {copy-lines="none",label="mode"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  annotations:
    crossplane.io/composition-schema-aware-validation-mode: loose
  # Removed for brevity
spec:
  # Removed for brevity
```

{{<hint "important" >}}验证模式也适用于由配置包定义的 Composition。

根据在 Composition 中配置的模式，模式识别验证问题可能会导致警告或拒绝接受 Composition。

查看 crossplane 日志，查看验证警告。

如果出现验证错误，crossplane 会将配置设为不健康状态。 使用 `kubectl describe configuration` 查看配置详细信息，以查看具体错误。{{< /hint >}}
