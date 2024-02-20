---

title: 管理资源
weight: 10
description: "托管资源是外部 Provider 资源的 crossplane 表示"

---

受管资源（`MR`）代表 Provider 中的外部服务。 当用户创建一个新的受管资源时，Provider 会在 Provider 环境中创建一个外部资源来做出反应。 Crossplane 管理的每个外部服务都会映射到一个受管资源。

{{< hint "note" >}}crossplane 将 Kubernetes 内部的对象称为_managed resource_，将 Provider 内部的外部对象称为_external resource_。{{< /hint >}}

管理资源的例子包括

* Amazon AWS EC2 [`实例`](https://marketplace.upbound.io/providers/upbound/provider-aws/latest/resources/ec2.aws.upbound.io/Instance/v1beta1)
* Google Cloud GKE [`集群`](https://marketplace.upbound.io/providers/upbound/provider-gcp/latest/resources/container.gcp.upbound.io/Cluster/v1beta1)
* Microsoft Azure PostgreSQL [`数据库`](https://marketplace.upbound.io/providers/upbound/provider-azure/latest/resources/dbforpostgresql.azure.upbound.io/Database/v1beta1)

{{< hint "tip" >}}

您可以创建单独的托管资源，但 crossplane 建议您使用 [Composition]({{<ref "./compositions" >}}) 和 claims 来创建托管资源。{{< /hint >}}

## 受管资源字段

Provider 定义受管资源的组别、种类和版本。 Provider 还定义受管资源的可用设置。

#### 组别、种类和版本

每个托管资源都是一个独特的 API 组端点，有自己的组、种类和版本。

例如，[Upbound AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-aws/latest/)定义了 {{<hover label="gkv" line="2">}}实例{{</hover>}}类中的 {{<hover label="gkv" line="1">}}ec2.aws.upbound.io{{</hover>}}

```yaml {label="gkv",copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
```

<!-- vale off -->

#### 删除政策

<!-- vale on -->

托管资源的 "删除策略 "会告诉 Provider 删除托管资源后要做什么。 如果 "删除策略 "是 "删除"，Provider 就会同时删除外部资源。 如果 "删除策略 "是 "orphan"，Provider 就会删除托管资源，但不会删除外部资源。

#### 选项

* 删除策略: Delete` - **默认** - 删除托管资源时删除外部资源。
* `deletionPolicy: Orphan` - 删除托管资源时保留外部资源。

#### 与管理政策的互动

在下列情况下，[管理策略](#managementpolicies) 优先于 "删除策略": 

<!-- vale write-good.Passive = NO -->

* 相关管理策略 alpha 功能已启用。

<!-- vale write-good.Passive = YES -->

* 该资源配置了默认值以外的管理策略。

详见下表。

{{< table "table table-sm table-hover">}}| managementPolicies | deletionPolicy | result | |-----------------------------|------------------|---------| | "_" (默认) | Delete (默认) | Delete | "_" (默认) | Orphan | Orphan | | 包含 "删除" | Delete (默认) | Delete | 包含 "删除" | Orphan | Delete | 不包含 "删除" | Delete (默认) | Orphan | 不包含 "删除" | Orphan | Orphan | 不包含 "删除" | Orphan | Orphan |{{< /table >}}

<!-- vale off -->

### 为提供者

<!-- vale on -->

规格 {{<hover label="forProvider" line="4">}}的{{</hover>}}会映射到外部资源的参数。

例如，在创建 AWS EC2 实例时，Provider 支持定义 AWS {{<hover label="forProvider" line="5">}}区域{{</hover>}}和虚拟机大小，称为{{<hover label="forProvider" line="6">}}实例类型{{</hover>}}.

{{< hint "note" >}}Provider 定义了设置及其有效值。 Provider 还在 `forProvider` 定义中定义了必填值和可选值。

详情请参阅特定 Provider 的文档。{{< /hint >}}

```yaml {label="forProvider",copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
# Removed for brevity
spec:
  forProvider:
    region: us-west-1
    instanceType: t2.micro
```

{{< hint "important">}}Crossplane 将托管资源的 "for Provider "字段视为外部资源的 "真实来源"。 Crossplane 会覆盖在 Crossplane 之外对外部资源所做的任何更改。 如果用户在 Provider 的网络控制台内进行更改，Crossplane 会将更改还原为 "for Provider "设置中的配置。{{< /hint >}}

#### 引用其他资源

受管资源中的某些字段可能依赖于其他受管资源中的 Values。 例如，虚拟机可能需要使用虚拟网络的名称。

托管资源可以通过外部名称、名称引用或选择器引用其他托管资源。

##### 按外部名称匹配

通过名称匹配资源时，crossplane 会在 Provider 中查找外部资源的名称。

例如，名为 "my-test-vpc "的 AWS VPC 对象的外部名称为 "vpc-01353cfe93950a8ff"。

```shell {copy-lines="1"
kubectl get vpc
NAME READY SYNCED EXTERNAL-NAME AGE
my-test-vpc True True vpc-01353cfe93950a8ff 49m
```

要按名称匹配 VPC，请引用外部名称。 例如，创建附加到此 VPC 的子网托管资源。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcId: vpc-01353cfe93950a8ff
```

##### 根据名称参考进行匹配

要根据管理资源的名称而不是 Provider 内部的外部资源名称来匹配资源，请使用 `nameRef`。

例如，名为 "my-test-vpc "的 AWS VPC 对象的外部名称为 "vpc-01353cfe93950a8ff"。

```shell {copy-lines="1"}
kubectl get vpc
NAME READY SYNCED EXTERNAL-NAME AGE
my-test-vpc True True vpc-01353cfe93950a8ff 49m
```

要通过名称引用匹配 VPC，请使用托管资源名称。 例如，创建附加到此 VPC 的子网托管资源。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcIdRef: 
      name: my-test-vpc
```

##### 按选择器匹配

通过选择器匹配是最灵活的匹配方法。

{{<hint "note" >}}

Composition]({{<ref "./compositions">}}) 部分涵盖了 `matchControllerRef` 选择器。{{</hint >}}

使用 "matchLabels "匹配被引用到资源的标签。 例如，此子网资源只匹配标签为 "my-label: label-value "的 VPC 资源。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcIdSelector: 
      matchLabels:
        my-label: label-value
```

#### 不可变字段

有些 Provider 不支持在创建后更改某些托管资源的字段。 例如，您不能更改 Amazon AWS `RDSInstance` 的 `region` 字段。 这些字段是_不可更改字段_。 Amazon 要求您删除并重新创建资源。

Crossplane 允许你编辑托管资源的不可变字段，但不会应用更改。 Crossplane 绝不会根据 "forProvider "更改删除资源。

{{<hint "note" >}}

<!-- vale write-good.Passive = NO -->

Crossplane 的行为与 Terraform 等其他工具不同。 Terraform 会删除并重新创建资源，以更改不可变字段。 Crossplane 只有在相应的托管资源对象从 Kubernetes 中删除且 "deletionPolicy "为 "delete "时，才会删除外部资源。

<!-- vale write-good.Passive = YES -->

{{< /hint >}}

#### 初始化延迟

Crossplane 默认将托管资源视为真实值来源；它希望拥有 `spec.forProvider` 下的所有值，包括可选值。 如果未提供，Crossplane 将使用提供商分配的值填充空字段。 例如，考虑 `region` 和 `availabilityZone` 等字段。 您可以只指定区域，让云提供商选择可用性区域。 在这种情况下，如果提供商分配了可用性区域，Crossplane 将使用该值填充 `spec.forProviders.availabilityZone` 字段。

{{<hint "note" >}}

<!-- vale write-good.Passive = NO -->

使用 [managementPolicies]({{<ref "./managed-resources#managementpolicies" >}})，可以通过在 "managementPolicies "列表中不包含 "LateInitialize "策略来关闭此行为。

<!-- vale write-good.Passive = YES -->

{{< /hint >}}

<!-- vale off -->

### initProvider

<!-- vale on -->

{{<hint "important" >}}托管资源 `initProvider` 选项是一个测试版功能，与 [managementPolicies]({{<ref "./managed-resources#managementpolicies" >}}).

{{< /hint >}}

启动{{<hover label="initProvider" line="7">}}initProvider{{</hover>}}定义了 Crossplane 仅在创建新托管资源时应用的设置。 Crossplane 会忽略在{{<hover label="initProvider" line="7">}}中定义的设置。{{</hover>}}字段中定义的设置。

{{<hint "note" >}}Crossplane始终会强制执行 "forProvider "中的设置。 Crossplane会恢复对外部资源中 "forProvider "字段的任何更改。

Crossplane不会强制执行 "initProvider "中的设置。 Crossplane会忽略外部资源中 "initProvider "字段的任何更改。{{</hint >}}

在设置 Provider 可能自动更改的初始 Values（如自动缩放组）时，使用 `initProvider` 非常有用。

例如，创建一个{{<hover label="initProvider" line="2">}}节点组{{</hover>}}的初始{{<hover label="initProvider" line="9">}}的节点组。{{</hover>}}的 NodeGroup 不会改变{{<hover label="initProvider" line="9">}}desiredSize{{</hover>}}设置。

{{< hint "tip" >}}crossplane 建议配置{{<hover label="initProvider" line="6">}}管理策略{{</hover>}}以避免与 `initProvider` 设置冲突。{{< /hint >}}

```yaml {label="initProvider",copy-lines="none"}
apiVersion: eks.aws.upbound.io/v1beta1
kind: NodeGroup
metadata:
  name: sample-eks-ng
spec:
  managementPolicies: ["Observe", "Create", "Update", "Delete"]
  initProvider:
    scalingConfig:
      - desiredSize: 1
  forProvider:
    region: us-west-1
    scalingConfig:
      - maxSize: 4
        minSize: 1
```

<!-- vale off -->

#### 管理政策

<!-- vale on -->

{{<hint "note" >}}受管资源 "managementPolicies "选项是测试版功能。 Crossplane 默认启用测试版功能。

请参阅 "提供方 "的文件，查看 "提供方 "是否支持管理政策。{{< /hint >}}

crossplane{{<hover label="managementPol1" line="4">}}管理策略{{</hover>}}决定 Crossplane 可对受管资源及其相应的外部资源采取哪些行动。 应用一个或多个{{<hover label="managementPol1" line="4">}}管理策略{{</hover>}}来确定 Crossplane 对资源的权限。

例如，要赋予 crossplane 创建和删除外部资源的权限，但不做任何更改，可将策略设置为{{<hover label="managementPol1" line="4">}}["创建"、"删除"、"观察"] .{{</hover>}}.

```yaml {label="managementPol1"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  managementPolicies: ["Create", "Delete", "Observe"]
  forProvider:
    # Removed for brevity
```

默认策略授予 crossplane 对资源的完全控制权。 使用空数组定义 `managementPolicies` 字段 [pauses]（#paused）资源。

{{<hint "important" >}}请参阅 "提供方 "的文件，查看 "提供方 "是否支持管理政策。{{< /hint >}}

Crossplane 支持以下策略: {{<table "table table-sm table-hover">}}| 策略 | 描述 | | --- | --- | | `*` | _默认策略_。 Crossplane 可以完全控制资源。 | | `Create` | 如果外部资源不存在，Crossplane 会根据托管资源的设置创建它。 | | `Delete` | Crossplane 可以在删除托管资源时删除外部资源。 | | `LateInitialize` | Crossplane 会初始化托管资源的 `spec.forProvider` 中没有定义的一些外部资源设置。 更多详情，请参阅 [late initialization]() 部分。{{<ref "./managed-resources#late-initialization" >}}更多详情，请参阅[the late initialization]()部分。 | | `Observe` | crossplane只观察资源，不做任何更改。 被引用到[observe only resources]({{<ref "/knowledge-base/guides/import-existing-resources#import-resources-automatically">}}| |`Update` | Crossplane 会在更改托管资源时更改外部资源。{{</table >}}

以下是常见策略组合的列表: {{<table "table table-sm table-hover table-striped-columns" >}}| Create | Delete | LateInitialize | Observe | Update | Description |
| :---:  | :---:  | :---:          | :---:   | :---:  | ---         |
| ✔️      | ✔️      | ✔️              | ✔️       | ✔️      | _Default policy_. Crossplane has full control over the resource.                                                                                                     |
| ✔️      | ✔️      | ✔️              | ✔️       |        | After creation any changes made to the managed resource aren't passed to the external resource. Useful for immutable external resources. |
| ✔️      | ✔️      |                | ✔️       | ✔️      | Prevent Crossplane from managing any settings not defined in the managed resource. Useful for immutable fields in an external resource. |
| ✔️      | ✔️      |                | ✔️       |        | Crossplane doesn't import any settings from the external resource and doesn't push changes to the managed resource. Crossplane recreates{{<ref "/knowledge-base/guides/import-existing-resources#import-resources-automatically">}}没有策略设置。 [暂停]（#paused）资源的替代方法。{{< /table >}}

<!-- vale off -->

### ProviderConfigRef

<!-- vale on -->

托管资源上的 `providerConfigRef` 会告诉 Provider 在创建托管资源时使用哪种 [ProviderConfig{{<ref "./providers#provider-configuration">}}在创建托管资源时被引用的 [ProviderConfigRef]。

使用 ProviderConfig 定义与 Provider 通信时使用的身份验证方法。

{{< hint "important" >}}如果未引用 `providerConfigRef`，Provider 会使用名为 `default` 的 ProviderConfig。{{< /hint >}}

例如，一个托管资源引用了名为{{<hover label="pcref" line="6">}}用户键{{</hover>}}.

这与 {{<hover label="pc" line="4">}}名称{{</hover>}}的名称。

```yaml {label="pcref",copy-lines="none"}}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
    # Removed for brevity
  providerConfigRef: user-keys
```

```yaml {label="pc"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: user-keys
# Removed for brevity
```

{{< hint "tip" >}}每个受管资源可以引用不同的 ProviderConfigs，这样不同的受管资源就可以使用不同的凭据对同一个 Provider 进行身份验证。{{< /hint >}}

<!-- vale off -->

### ProviderRef

<!-- vale on -->

<!-- vale Crossplane.Spelling = NO -->

在 `crossplane-runtime` [v0.10.0](https://github.com/crossplane/crossplane-runtime/releases/tag/v0.10.0) 中，crossplane 废弃了 `providerRef` 字段。使用 `providerRef` 的托管资源必须使用 [`providerConfigRef`](#providerconfigref)。

<!-- vale Crossplane.Spelling = YES -->

<!-- vale off -->

### writeConnectionSecretToRef

<!-- vale on -->

Provider 创建受管资源时，可能会生成特定于资源的详细信息，如用户名、密码或 IP 地址等连接详细信息。

crossplane 会将这些详细信息存储在由 `writeConnectionSecretToRef` 值指定的 Kubernetes Secret 对象中。

例如，当用 crossplane [community AWS provider](https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws/v0.40.0)创建 AWS RDS 数据库实例时，会生成端点、密码、端口和用户名数据。 Provider 会将这些变量保存在 Kubernetes secret 中{{<hover label="secretname" line="9" >}}rds-secret{{</hover>}}中，由{{<hover label="secretname" line="9" >}}writeConnectionSecretToRef{{</hover>}}字段引用。

```yaml {label="secretname",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance
spec:
  forProvider:
  # Removed for brevity
  writeConnectionSecretToRef:
    name: rds-secret
```

查看 Secret 对象会显示保存的字段。

```yaml {copy-lines="1"}
kubectl describe secret rds-secret
Name:         rds-secret
# Removed for brevity
Data
====
port:      4 bytes
username:  10 bytes
endpoint:  54 bytes
password:  27 bytes
```

{{<hint "important" >}}Provider 决定写入 Secret 对象的数据。 有关生成的 Secret 数据，请参阅特定 Provider 文档。{{< /hint >}}

<!-- vale off -->

### PublishConnectionDetailsTo

<!-- vale on -->

publishConnectionDetailsTo "字段扩展了[`writeConnectionSecretToRef`](#writeconnectionsecrettoref)，支持将托管资源信息存储为 Kubernetes Secret 对象或 [Vault](https://www.vaultproject.io/) 等外部存储。

使用 "publishConnectionDetailsTo "需要启用 Crossplane 外部存储（ESS）。 在 Provider 中使用 [DeploymentRuntimeConfig]() 启用 ESS，在 Crossplane 中使用"--enable-external-secret-stores "参数启用 ESS。{{<ref "providers#runtime-configuration" >}}) 并在 crossplane 中使用 `--enable-external-secret-stores` 参数启用 ESS。

{{< hint "note" >}}并非所有 Provider 都支持 "publishConnectionDetailsTo"，详情请查看 Provider 文档。{{< /hint >}}

#### 向 Kubernetes 发布secret

要将托管资源生成的数据发布为 Kubernetes Secret 对象，请提供一个{{<hover label="k8secret" line="7">}}publishConnectionDetailsTo.name{{< /hover >}}

```yaml {label="k8secret",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
```

crossplane 还可以通过使用{{<hover label="k8label" line="8">}}publishConnectionDetailsTo.metadata{{</hover>}}.

```yaml {label="k8label",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
    metadata:
      labels:
        label-tag: label-value
      annotations:
        annotation-tag: annotation-value
```

#### 向外部secret存储发布secret

向外部secret存储（如 [HashiCorp Vault](https://www.vaultproject.io/)）发布secret数据依赖于一个{{<hover label="configref" line="8">}}publishConnectionDetailsTo.configRef{{</hover>}}.

配置{{<hover label="configref" line="9">}}configRef.name{{</hover>}}引用了一个{{<hover label="storeconfig" line="4">}}存储配置{{</hover>}}对象。

```yaml {label="configref",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
    configRef: 
      name: my-vault-storeconfig
```

```yaml {label="storeconfig",copy-lines="none"}
apiVersion: secrets.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: my-vault-storeconfig
# Removed for brevity
```

{{<hint "tip" >}}请阅读[Vault 作为外部secret存储]({{<ref "knowledge-base/integrations/vault-as-secret-store">}}) 指南，了解使用 StoreConfig 对象的详情。{{< /hint >}}

## Annotations

crossplane 将一套标准的 Kubernetes "Annotations "应用于托管资源。

{{<table "table table-sm">}}| Annotation | Definition | 
| --- | --- | 
| `crossplane.io/external-name` | The name of the managed resource inside the Provider. |
| `crossplane.io/external-create-pending` | The timestamp of when Crossplane began creating the managed resource. | 
| `crossplane.io/external-create-succeeded` | The timestamp of when the Provider successfully created the managed resource. | 
| `crossplane.io/external-create-failed` | The timestamp of when the Provider failed to create the managed resource. | 
| `crossplane.io/paused` | Indicates Crossplane isn't reconciling this resource. Read the [Pause Annotation](#paused) for more details. |
| `crossplane.io/composition-resource-name` | For managed resource created by a Composition, this is the Composition's `resources.name` value. |{{</table >}}

### 命名外部资源

默认情况下，Provider 会将外部资源命名为与 Kubernetes 对象相同的名称。

例如，一个名为{{<hover label="external-name" line="4">}}my-rds-instance{{</hover >}}的托管资源的名称为 "my-rds-instance"，作为 Provider 环境中的外部资源。

```yaml {label="external-name",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance
```

```shell
kubectl get rdsinstance
NAME READY SYNCED EXTERNAL-NAME AGE
my-rds-instance True True my-rds-instance 11m
```

使用已提供的 "crossplane.io/external-name "注解创建的托管资源会使用注解值作为外部资源名称。

例如，Provider 创建了名为{{< hover label="custom-name" line="6">}}my-rds-instance{{</hover>}}但被引用为 {{<hover label="custom-name" line="5">}}我的自定义名称{{</hover >}}作为 AWS 内部的外部资源。

```yaml {label="custom-name",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance  
  annotations: 
    crossplane.io/external-name: my-custom-name
```

```shell {copy-lines="1"}
kubectl get rdsinstance
NAME READY SYNCED EXTERNAL-NAME AGE
my-rds-instance True True my-custom-name 11m
```

#### 创建 Annotations

当 AWS 等外部系统生成不确定的资源名称时，Provider 就有可能创建了一个资源，但却没有记录。 出现这种情况时，Provider 就无法管理该资源。

{{<hint "tip">}}crossplane 将 Provider 创建但不管理的资源称为_leaked resources_。{{</hint>}}

Providers 设置了三个创建 Annotations，以避免和检测泄漏的资源: 

* {{<hover label="creation" line="8">}}crossplane.io/external-create-pending{{</hover>}} - Providers 上次准备创建资源的时间。
* {{<hover label="creation" line="9">}}crossplane.io/external-create-succeeded{{</hover>}} - Provider 上次成功创建资源的时间。
* crossplane.io/external-create-failed` - 上次 Providers 创建资源失败的时间。

使用 `kubectl get` 查看受管资源上的 annotations。 例如，AWS VPC 资源: 

```yaml {label="creation" copy-lines="2-9"}
$ kubectl get -o yaml vpc my-vpc
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: my-vpc
  annotations:
    crossplane.io/external-name: vpc-1234567890abcdef0
    crossplane.io/external-create-pending: "2023-12-18T21:48:06Z"
    crossplane.io/external-create-succeeded: "2023-12-18T21:48:40Z"
```

一个 Provider 会被引用{{<hover label="creation" line="7">}}crossplane.io/external-name{{</hover>}}Annotations 来查找外部系统中的托管资源。

Providers 会在外部系统中查找资源，以确定它是否存在，以及是否与托管资源的期望状态相匹配。 如果找不到资源，Providers 就会创建它。

有些外部系统不允许 Providers 在创建资源时指定资源名称，而是由外部系统生成一个非确定名称并返回给 Providers。

当外部系统生成资源名称时，Provider 会尝试将其保存到受管资源的 `crossplane.io/external-name` 注解中。 如果没有，它就会_泄露_资源。

Providers 无法保证能保存 Annotations。 Providers 可能会在创建资源和保存注释之间重新启动或失去网络连接。

Providers 可以检测到自己可能泄露了资源。 如果 Providers 认为自己可能泄露了资源，它就会停止对账，直到您告诉 Providers 可以安全继续。

{{<hint "important">}}只要外部系统生成了资源名称，Providers 就有可能泄露资源。

当 Providers 检测到自己可能泄露了资源时，最安全的做法就是停下来等待人工干预。

这样可以确保 Providers 不会重复创建泄漏的资源。 重复的资源可能代价高昂而且危险。{{</hint>}}

当 Provider 认为自己可能泄露了资源时，它会创建一个与托管资源相关的 "无法确定创建结果 "事件。 被引用 "kubectl describe "可查看该事件。

```shell {copy-lines="1"}
kubectl describe queue my-sqs-queue

# Removed for brevity

Events:
  Type Reason Age From Message
  ----     ------                           ----                ----                                 -------
  Warning CannotInitializeManagedResource 29m (x19 over 19h)  managed/queue.sqs.aws.crossplane.io cannot determine creation result - remove the crossplane.io/external-create-pending annotation if it is safe to proceed
```

Provider 使用创建 Annotations 来检测自己是否可能泄露了资源。

Providers 每次核对托管资源时都会检查资源的创建 Annotations。 如果 Providers 看到创建等待时间比最近的创建成功或创建失败时间更近，它就知道自己可能泄露了资源。

{{<hint "note">}}Providers 不会删除创建注解。 他们会使用时间戳来确定哪个是最新的。 一个托管资源有几个创建注解是很正常的。{{</hint>}}

Provider 知道自己可能泄露了资源，因为它会同时更新资源的所有 Annotations。 如果 Provider 在创建资源后无法更新创建注解，那么它也无法更新 `crossplane.io/external-name` 注解。

{{<hint "tip">}}如果资源出现 "无法确定创建结果 "错误，请检查外部系统。

使用来自 `crossplane.io/external-create-pending` 注解的时间戳来确定 Providers 可能泄露资源的时间。 查找在此时间前后创建的资源。

如果发现泄漏的资源，而且安全的话，请将其从外部系统中删除。

在确定不存在泄漏资源后，从托管资源中移除 "crossplane.io/external-create-pending "注解。 这将告诉 Providers 恢复调节并重新创建托管资源。{{</hint>}}

Provider 也会使用创建 Annotations 来避免资源泄露。

当 Providers 写入 "crossplane.io/external-create-pending "注解时，它知道自己正在调和最新版本的托管资源。 如果 Providers 正在调和旧版本的托管资源，写入将失败。

如果 Providers 将旧版本与过时的 `crossplane.io/external-name` 注解进行核对，它可能会错误地判定该资源不存在。 Providers 会创建一个新资源，并泄漏现有资源。

有些外部系统在 Providers 创建资源和系统报告资源存在之间会有延迟。 Providers 会使用最近的创建成功时间来解释这一延迟。

如果 Providers 没有考虑到延迟，它可能会错误地判断该资源不存在。 Providers 会创建一个新资源，并泄漏现有资源。

#### 暂停

手动应用 `crossplane.io/paused` 注解会导致 Provider 停止调节托管资源。

在修改 Provider 或编辑 Kubernetes 对象时，暂停资源非常有用。

apply a {{<hover label="pause" line="6">}}crossplane.io/paused: "true"{{</hover>}}注释来暂停调和托管资源。

{{< hint "note" >}}只有值`"true"`才会暂停调和。{{< /hint >}}

```yaml {label="pause"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: my-rds-instance
  annotations:
    crossplane.io/paused: "true"
spec:
  forProvider:
    region: us-west-1
    instanceType: t2.micro
```

删除 Annotations，恢复对账。

## Finalizer

crossplane 对托管资源应用 [Finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) 来控制其删除。

{{< hint "note" >}}Kubernetes 无法使用 Finalizer 删除对象。{{</hint >}}

当 crossplane 删除受管资源时，Provider 会开始删除外部资源，但受管资源仍会保留，直到外部资源被完全删除。

外部资源完全删除后，crossplane 会移除 Finalizer 并删除托管资源对象。

## 条件

crossplane 有一套标准的受管资源 "条件 "集。 使用 `kubectl describe<managed_resource>` 查看受管资源的 "条件"。

{{<hint "note" >}}提供商可以定义自己的自定义 "条件"。{{</hint >}}

### 可用

Reason: Available "表示 Provider 创建了托管资源，并已准备好供使用。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                True
  Reason:                Available
```

#### 创建

原因: 正在创建 "表示 Provider 正在尝试创建托管资源。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Creating
```

#### 删除

原因: 正在删除 "表示 Provider 正在尝试删除托管资源。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Deleting
```

<!-- vale off -->

### ReconcilePaused

<!-- vale on -->

Reason:ReconcilePaused "表示受管资源有[Pause]（#paused）注解

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                False
  Reason:                ReconcilePaused
```

<!-- vale off -->

### ReconcileError

<!-- vale on -->

原因: ReconcileError "表示 Crossplane 在调节托管资源时遇到错误。 条件 "的 "Message: "值有助于识别 Crossplane 的错误。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                False
  Reason:                ReconcileError
```

<!-- vale off -->

### ReconcileSuccess

<!-- vale on -->

Reason:ReconcileSuccess "表示 Provider 创建并正在监控托管资源。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                True
  Reason:                ReconcileSuccess
```

### 无法提供

Reason:Unavailable"（原因: 不可用）表示 crossplane 预计受管资源可用，但 Provider 报告该资源不健康。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Unavailable
```

### 未知

`Reason: Unknown` 表示 Provider 与托管资源发生了意外错误。 `conditions.message`提供了出错原因的更多信息。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Unknown
  Status:                False
  Reason:                Unknown
```

### Upjet Providers 条件

[Upjet](https://github.com/upbound/upjet)是生成 Crossplane Providider 的开源工具，它也有一套标准的 "条件"。

<!-- vale off -->

#### AsyncOperation

<!-- vale on -->

有些资源的创建时间可能超过一分钟，基于 Upjet 的 Provider 可以通过异步操作，在创建托管资源前完成 Kubernetes 命令。

##### 已完成

原因: 已完成 "表示异步操作已成功完成。

```yaml {copy-lines="none"}
Conditions:
  Type:                  AsyncOperation
  Status:                True
  Reason:                Finished
```

##### 进行中

原因: 正在进行 "表示受管资源操作仍在进行中。

```yaml {copy-lines="none"}
Conditions:
  Type:                  AsyncOperation
  Status:                True
  Reason:                Ongoing
```

<!-- vale off -->

#### LastAsyncOperation

<!-- vale on -->

Upjet `Type: LastAsyncOperation` 捕获上一次异步操作的状态，即 "成功 "或失败 "原因"。

<!-- vale off -->

##### ApplyFailure

<!-- vale on -->

`Reason: ApplyFailure` 表示 Provider 向托管资源应用设置失败。 `conditions.message`提供了更多有关出错原因的信息。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                False
  Reason:                ApplyFailure
```

<!-- vale off -->

##### DestroyFailure

<!-- vale on -->

`Reason: DestroyFailure` 表示 Provider 删除托管资源失败。 `conditions.message`提供了更多关于出错原因的信息。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                False
  Reason:                DestroyFailure
```

##### 成功

原因: 成功 "表示 Provider 成功地异步创建了托管资源。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                True
  Reason:                Success
```