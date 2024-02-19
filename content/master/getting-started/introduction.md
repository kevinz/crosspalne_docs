---

title: crossplane 简介
weight: 2

---

crossplane 可将 Kubernetes 集群连接到外部非 Kubernetes 资源，并允许平台团队构建定制的 Kubernetes API 来使用这些资源。

<!-- vale gitlab.SentenceLength = NO -->

Crossplane 会创建 Kubernetes [Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) (`CRDs`)，将外部资源表示为本地 [Kubernetes 对象](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)。作为本地 Kubernetes 对象，你可以使用`kubectl create` 和`kubectl describe` 等标准命令。每个 Crossplane 资源都可以使用完整的 [Kubernetes API](https://kubernetes.io/docs/reference/using-api/)。

<!-- vale gitlab.SentenceLength = YES -->

Crossplane 还充当[Kubernetes 控制器](https://kubernetes.io/docs/concepts/architecture/controller/)，观察外部资源的状态并提供状态执行。如果有东西修改或删除了 Kubernetes 外部的资源，Crossplane 会逆转更改或重新创建被删除的资源。

{{<img src="/media/crossplane-intro-diagram.png" alt="Diagram showing a user communicating to Kubernetes. Crossplane connected to Kubernetes and Crossplane communicating with AWS, Azure and GCP" align="center">}}在Kubernetes集群中安装Crossplane后，用户只与Kubernetes通信，Crossplane则管理与外部资源（如AWS、Azure或谷歌云）的通信。

crossplane 还允许创建自定义 Kubernetes API。 平台团队可以结合外部资源，简化或自定义呈现给平台消费者的 API。

## crossplane 组件概览

本表概述了 crossplane 组件及其作用。

{{< table "table table-hover table-sm">}}| 组件 | 缩写 | 范围 | 摘要 | | --- | --- | ---- | | [Provider]({{<ref "#providers">}}) | | 集群 | 为外部服务创建新的 Kubernetes 自定义资源定义。{{<ref "#provider-configurations">}}) | `PC` | 集群 | 应用_Provider_的设置。{{<ref "#managed-resources">}}) | | `MR` | 集群 | 由 Crossplane 在 Kubernetes 集群内创建和管理的 Provider 资源。{{<ref "#compositions">}}) | | 集群 | 用于一次性创建多个_托管资源_的模板。{{<ref "#composite-resources" >}}) | `XR` | 集群 | 使用_Composition_模板将多个_managed resources_创建为一个Kubernetes对象。{{<ref "#composite-resource-definitions" >}}) | `XRD` | 集群 | 定义_复合资源_和_claim_的 API 模式 | | | [Claims]({{<ref "#claims" >}}) | `XC` | namespace | 类似于 _Composite Resource_，但作用域为 namespace。{{< /table >}}

## The Crossplane Pod

Crossplane 安装在 Kubernetes 集群中时，会创建一组 Crossplane 核心组件的初始自定义资源定义（CRD）。

{{< expand "View the initial Crossplane CRDs" >}}安装 Crossplane 后，使用 `kubectl get crds` 查看已安装的 Crossplane CRD。

```shell
❯ kubectl get crd
NAME                                                    
compositeresourcedefinitions.apiextensions.crossplane.io
compositionrevisions.apiextensions.crossplane.io        
compositions.apiextensions.crossplane.io                
configurationrevisions.pkg.crossplane.io                
configurations.pkg.crossplane.io                        
controllerconfigs.pkg.crossplane.io                     
deploymentruntimeconfigs.pkg.crossplane.io              
environmentconfigs.apiextensions.crossplane.io          
functionrevisions.pkg.crossplane.io                     
functions.pkg.crossplane.io                             
locks.pkg.crossplane.io                                 
providerrevisions.pkg.crossplane.io                     
providers.pkg.crossplane.io                             
storeconfigs.secrets.crossplane.io                      
usages.apiextensions.crossplane.io
```

{{< /expand >}}

下文将介绍其中一些 CRD 的功能。

## Providers

Crossplane _Provider_创建了第二套CRD，定义了Crossplane如何连接到非Kubernetes服务。 每个外部服务都依赖于自己的Provider。例如，[AWS](https://marketplace.upbound.io/providers/upbound/provider-aws)、[Azure](https://marketplace.upbound.io/providers/upbound/provider-azure)和[GCP](https://marketplace.upbound.io/providers/upbound/provider-gcp)是每个云服务的不同提供商。

{{< hint "tip" >}}大多数 Provider 是针对云服务的，但 crossplane 可以使用 Provider 连接到任何具有 API 的服务。{{< /hint >}}

例如，AWS Provider 为 EC2 计算实例或 S3 存储桶等 AWS 资源定义 Kubernetes CRD。

Provider 定义了外部资源的 Kubernetes API 定义。例如，[Upbound Provider AWS](https://marketplace.upbound.io/providers/upbound/provider-aws/) 定义了用于创建和管理 AWS S3 存储桶的 [`bucket`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1)资源。

在`bucket`CRD中，有一个[`spec.forProvider.region`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1#doc:spec-forProvider-region)值，用于定义在哪个AWS区域部署bucket。

Upbound Marketplace 包含大量[crossplane Providers 集合](https://marketplace.upbound.io/providers)。

更多 Provider 可在 [Crossplane Contrib 资源库](https://github.com/crossplane-contrib/) 中找到。

Provider 具有集群作用域，可用于所有集群名称空间。

使用命令 `kubectl get providers` 查看所有已安装的 Provider。

## Provider 配置

_ProviderConfigs_ 配置与 Provider 相关的设置，如身份验证或 Provider 的全局默认值。

ProviderConfigs 的 API 端点对每个 Provider 都是唯一的。

_ProviderConfigs_ 具有集群作用域，可用于所有集群名称空间。

使用命令 `kubectl get providerconfig` 查看所有已安装的 ProviderConfigs。

## 受托管的资源

Provider 的 CRD 映射到 Provider 内部的单个 _资源_。 当 crossplane 创建并监控一个资源时，它就是一个 _受管资源_。

使用 Provider 的 CRD 会创建一个唯一的 _Managed Resource_。 例如，使用 AWS 的 `bucket` CRD，crossplane 会在 Kubernetes 集群内创建一个连接到 AWS S3 存储桶的 `bucket` _Managed Resource_。

Crossplane 控制器为 _Managed Resources_ 提供状态执行。 Crossplane 执行 _Managed Resources_ 的设置和存在。这种 "控制器模式 "就像 Kubernetes [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) 为 pod 执行状态一样。

_托管资源_ 具有集群作用域，可用于所有集群名称空间。

使用 `kubectl get managed` 查看所有_受管资源_。{{<hint "warning" >}}`kubectl get managed` 会创建大量 Kubernetes API 查询。 `kubectl` 客户端和 kube-apiserver 都会限制 API 查询。

根据 API 服务器的大小和托管资源的数量，该命令可能需要几分钟才能返回，也可能会超时。

更多信息，请阅读 [Kubernetes issue #111880](https://github.com/kubernetes/kubernetes/issues/111880) 和 [Crossplane issue #3459](https://github.com/crossplane/crossplane/issues/3459) 。{{< /hint >}}

## Compositions

组件允许平台团队将一组_托管资源_定义为单一对象。

例如，一个计算_托管资源_可能需要创建一个存储资源和一个虚拟网络。 一个_组成_对象可以在一个_组成_对象中定义所有这三种资源。

使用 _Compositions_ 可简化由多个_managed resources_组成的基础架构的部署。

平台团队可为_Composition_内的每个_managed resource_定义固定或默认设置，或定义用户可更改的字段和设置。

使用前面的例子，平台团队可能会设置计算资源大小和虚拟网络设置。 但平台团队允许用户定义存储资源大小。

创建 _Composition_ crossplane 并不创建任何受管资源。 _Composition_ 只是一个模板，用于集合 _managed resources_ 及其设置。 _Composite Resource_ 创建特定资源。

{{< hint "note" >}}[_复合资源_]({{<ref "#composite-resources">}}) 部分讨论了_复合资源_。{{< /hint >}}

_Compositions_ 具有集群作用域，可用于所有集群名称空间。

使用 `kubectl get compositions` 查看所有_compositions_。

## Composition Resources

一个 _Composite Resource_ (`XR`)是一组已调配的_managed resources_。 一个 _Composite Resource_ 使用一个 _Composition_ 所定义的模板，并应用任何用户定义的设置。

多个唯一的 _Composite Resource_ 对象可以使用同一个 _Composition_ 。 例如，一个 _Composition_ 模板可以创建一组计算、存储和网络的 _managed resources_。 每次用户请求这组资源时，crossplane 都会引用同一个 _Composition_ 模板。

如果 _Composition_ 允许用户定义资源设置，用户就会在 _Composite Resource_ 中应用这些设置。

<!-- A _Composition_ defines which _Composite Resources_ can use the _Composition_
template with the _Composition_ `spec.compositeTypeRef` value. This defines the
{{<hover label="comp" line="7">}}apiVersion{{< /hover >}} and {{<hover
label="comp" line="8">}}kind{{< /hover >}} of _Composite Resources_ that can use the
_Composition_.

For example, in the _Composition_:
```yaml {label="comp"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: test.example.org
spec:
  compositeTypeRef:
    apiVersion: test.example.org/v1alpha1
    kind: myComputeResource
    # Removed for brevity
```

A _Composite Resource_ that can use this template must match this 
{{<hover label="comp" line="7">}}apiVersion{{< /hover >}} and {{<hover
label="comp" line="8">}}kind{{< /hover >}}.

```yaml {label="xr"}
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

The _Composite Resource_ {{<hover label="xr" line="1">}}apiVersion{{< /hover >}}
matches the and _Composition_ 
{{<hover label="comp" line="7">}}apiVersion{{</hover >}} and the 
_Composite Resource_  {{<hover label="xr" line="2">}}kind{{< /hover >}}
matches the _Composition_ {{<hover label="comp" line="8">}}kind{{< /hover >}}.

In this example, the _Composite Resource_ also sets the 
{{<hover label="xr" line="7">}}storage{{< /hover >}} setting. The
_Composition_ uses this value when creating the associated _managed resources_
owned by this _Composite Resource_. -->

{{< hint "tip" >}}_Compositions_ 是一组_managed resources_ 的模板。_Composite Resources_ 填充模板并创建_managed resources_。

删除_复合资源_会删除它创建的所有_托管资源_。{{< /hint >}}

_Composite Resources_ 具有集群作用域，可用于所有集群名称空间。

使用 `kubectl get composite` 查看所有 _Composite 资源。

### 复合资源定义

_Composite Resource Definitions_ (`XRDs`)创建了_Claims_和_Composite Resources_所引用的自定义 Kubernetes API。

{{< hint "note" >}}claim]({{<ref "#claims">}}) 部分讨论了_claim_。{{< /hint >}}

平台团队可以定义自定义 API。 这些 API 可以定义以千兆字节为单位的存储空间等特定值、"小 "或 "大 "等通用设置、"云 "或 "onprem "等部署选项。 crossplane 不会限制 API 的定义。

_Composite Resource Definition 的_ `kind` 来自 crossplane。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
```

复合资源定义_的 `spec` 创建了_复合资源_的 `apiVersion`、 `kind` 和 `spec`。

{{< hint "tip" >}}复合资源定义_定义了_复合资源_的参数。{{< /hint >}}

一个 _Composite Resource Definition_ 有四个主要的 `spec` 参数: 

* A {{<hover label="specGroup" line="3" >}}组{{< /hover >}}

来定义{{< hover label="xr2" line="2" >}}apiVersion{{</hover >}}中定义 apiVersion。

* 版本.名称 {{< hover label="specGroup" line="7" >}}版本.名称{{</hover >}}

定义了_复合资源_中被引用的版本。

* A {{< hover label="specGroup" line="5" >}}names.kind{{</hover >}}

来定义_复合资源_______。{{< hover label="xr2" line="3" >}}种类{{</hover>}}.

* A {{< hover label="specGroup" line="8" >}}版本模式{{</hover>}}部分

来定义_复合资源_。 {{<hover label="xr2" line="6" >}}规格{{</hover >}}.

```yaml {label="specGroup"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  versions:
  - name: v1alpha1
    schema:
      # Removed for brevity
```

基于该_复合资源定义_的_复合资源_看起来像这样: 

```yaml {label="xr2"}
# Composite Resource (XR)
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

复合资源定义 {{< hover label="specGroup" line="8" >}}模式{{</hover >}}定义了_复合资源_的{{<hover label="xr2" line="6" >}}规格{{</hover >}}参数。

这些参数是新的、自定义的 API，开发人员可以使用。

例如，创建计算_托管资源_需要了解云 Provider 的计算类名称，如 AWS 的 `m6in.large` 或 GCP 的 `e2-standard-2`。

_Composite Resource Definition_ 可将选项限制为 "小 "或 "大"。 _Composite Resource_ 被引用这些选项，而 _Composition_ 则将其映射到特定的云提供商设置。

下面的_复合资源定义_定义了一个 {{<hover label="specVersions" line="17" >}}存储空间{{< /hover >}}存储空间是一个{{<hover label="specVersions" line="18">}}字符串{{< /hover >}}和 OpenAPI{{<hover label="specVersions" line="19" >}}的{{< /hover >}}要求选项必须是 {{<hover label="specVersions" line="20" >}}小{{< /hover >}}或 {{<hover label="specVersions" line="21" >}}大{{< /hover >}}.

```yaml {label="specVersions"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
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
              storage:
                type: string
                oneOf:
                  - pattern: '^small$'
                  - pattern: '^large$'
            required:
            - storage
```

一个_复合资源定义_可以定义多种设置和选项。

创建_复合资源定义_可以创建_复合资源_，但也可以创建_claim_。

带有 `spec.claimNames` 的 _Composite Resource Definition_ 允许开发人员创建 _Claims_。

例如{{< hover label="xrdClaim" line="6" >}}claimNames.kind{{</hover >}}允许创建 `kind: computeClaim` 的 _Claims。

```yaml {label="xrdClaim"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  claimNames:
    kind: computeClaim
  # Removed for brevity
```

## claims

claim是开发人员与 crossplane 交互的主要方式。

_Claims_ 访问平台团队在 _Composite Resource Definition_ 中定义的自定义应用程序接口。

_Claims_ 看起来像 _Composite Resources_，但它们是 namespace 作用域，而 _Composite Resources_ 是集群作用域。

{{< hint "note" >}}
**为什么名称空间范围很重要？** 具有名称空间范围的 _Claims_ 允许多个团队使用唯一的名称空间创建相同类型的资源，且相互独立。 团队 A 的计算资源与团队 B 的计算资源是唯一的。

直接创建 _Composite Resources_ 需要集群范围内的权限，并与所有团队共享。_Claims_ 创建相同的资源集，但在 namespace 层面上。{{< /hint >}}

之前的_复合资源定义_允许创建_claim_类型的{{<hover label="xrdClaim2" line="7" >}}computeClaim{{</hover>}}.

claim被引用相同的{{< hover label="xrdClaim2" line="3" >}}apiVersion{{< /hover >}}中定义的 apiVersion，_Composite Resource Definition_ 中定义的 apiVersion 也被_Composite Resources_ 引用。

```yaml {label="xrdClaim2"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  claimNames:
    kind: computeClaim
  # Removed for brevity
```

在 _Claim_ 的示例中{{<hover label="claim" line="2">}}apiVersion{{< /hover >}}与 {{<hover label="xrdClaim2" line="3">}}组{{< /hover >}}中的组。

claim {{<hover label="claim" line="3">}}类型{{< /hover >}}与_复合资源定义_匹配。{{<hover label="xrdClaim2" line="7">}}claimNames.kind{{< /hover >}}.

```yaml {label="claim"}
# Claim
apiVersion: test.example.org/v1alpha1
kind: computeClaim
metadata:
  name: myClaim
  namespace: devGroup
spec:
  size: "large"
```

一个 _Claim_ 可以安装在一个 {{<hover label="claim" line="6">}}namespace 中。{{</hover >}}复合资源定义_定义了{{<hover label="claim" line="7">}}规格{{< /hover >}}选项的方式与_复合资源_的方式相同{{<hover label="xr-claim" line="6">}}规格{{< /hover >}}.

{{< hint "tip" >}}_Composite Resources_ 和 _Claims_ 是相似的。 只有 _Claims_ 可以在一个命名空间中。 {{<hover label="claim" line="6">}}名称空间。{{</hover >}}此外，_复合资源_的 {{<hover label="xr-claim"
line="3">}}种类{{</hover >}}可能不同于_Claim_的{{<hover label="claim" line="3">}}种类。{{< /hover >}}_Composite Resource Definition_ 定义了{{<hover label="xrdClaim2" line="7">}}种类{{</hover >}}Values 的值。{{< /hint >}}

```yaml {label="xr-claim"}
# Composite Resource (XR)
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

_Claims_ 是 namespace 作用域。

使用命令 `kubectl get claim` 查看所有可用的 claims。

## 下一步

使用其中一个快速入门指南，构建自己的 crossplane 平台。

* [Azure 快速入门]({{<ref "provider-azure" >}})
* [AWS 快速入门]({{<ref "provider-aws" >}})
* [GCP 快速入门]({{<ref "provider-gcp" >}})
