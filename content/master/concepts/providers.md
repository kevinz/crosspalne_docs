---

title: Providers
weight: 5
description: "Providers（提供者）将 crossplane 连接到外部 API"

---

Providers 支持 crossplane 在外部服务上配置基础设施。 Providers 创建新的 Kubernetes API 并将其映射到外部 API。

Provider 负责连接到非 Kubernetes 资源的所有方面。 这包括身份验证、进行外部 API 调用以及为任何外部资源提供 [Kubernetes Controller](https://kubernetes.io/docs/concepts/architecture/controller/) 逻辑。

Providers 的例子包括

* [Provider AWS](https://github.com/upbound/provider-aws)
* [提供商 Azure](https://github.com/upbound/provider-azure)
* [提供商 GCP](https://github.com/upbound/provider-gcp)
* [提供商 Kubernetes](https://github.com/crossplane-contrib/provider-kubernetes)

{{< hint "tip" >}}在 [Upbound Marketplace](https://marketplace.upbound.io) 中查找更多 Provider。{{< /hint >}}

<!-- vale write-good.Passive = NO -->

<!-- "are Managed" isn't passive in this context -->

Provider 会把他们能在 Kubernetes 中创建的每个外部资源定义为 Kubernetes API 端点。 这些端点是 [_Managed Resources_]({{<ref "managed-resources" >}}).

<!-- vale write-good.Passive = YES -->

## 安装一个 Provider

安装一个 Provider 会创建代表 Provider API 的新 Kubernetes 资源。 安装一个 Provider 还会创建一个 Provider pod，负责将 Provider 的 API 调节到 Kubernetes 集群中。 Provider 会持续观察所需托管资源的状态，并创建任何缺失的外部资源。

使用crossplane安装 Provider{{<hover label="install" line="2">}}Provider{{</hover >}}对象，设置{{<hover label="install" line="6">}}spec.packages{{</hover >}}值为 Provider 软件包的位置。

例如，安装 [AWS Community Provider](https://github.com/crossplane-contrib/provider-aws)、

```yaml {label="install"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0
```

默认情况下，Provider pod 会安装在与 crossplane (`crossplane-system`) 相同的命名空间中。

{{<hint "note" >}}Providers 是{{<hover label="install" line="1">}}pkg.crossplane.io{{</hover>}}组的一部分。

.meta.pkg.crossplane.io。 {{<hover label="meta-pkg" line="1">}}meta.pkg.crossplane.io{{</hover>}}组用于创建 Provider 软件包。

有关构建 Provider 的说明不在本文档的范围之内。 请阅读跨plane 贡献的 [Provider 开发指南](https://github.com/crossplane/crossplane/blob/master/contributing/guide-provider-development.md) 了解更多信息。

有关 Provider 程序包规范的信息，请阅读 [Crossplane Provider Package specification](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md#provider-package-requirements)。

```yaml {label="meta-pkg"}
apiVersion: meta.pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
# Removed for brevity
```

{{</hint >}}

### 用 Helm 安装

Crossplane 支持在初始安装 Crossplane 时使用 Crossplane Helm chart 安装 Providers。

被引用{{<hover label="helm" line="5" >}}--set provider.packages{{</hover >}}参数与 `helm install` 一起使用。

例如，安装 AWS Community Provider、

```shell {label="helm"}
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--set provider.packages='{xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0}'
```

#### 离线安装

crossplane 从本地软件包缓存中安装软件包。默认情况下，crossplane 软件包缓存是一个 [emptyDir volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)。

配置 Crossplane，使其被引用[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，以使用包含 Provider 镜像的存储位置。阅读[Crossplane 安装文档]( ) 中有关配置 Crossplane Pod 设置的更多信息。{{<ref "../software/install#customize-the-crossplane-helm-chart">}}).

提供 Provider 的`.xpkg`文件名并设置{{<hover label="offline" line="7">}}packagePullPolicy: Never{{</hover>}}.

例如，要安装本地存储的 Provider AWS 版本，请将{{<hover label="offline" line="6">}}package{{</hover>}}为本地文件名，并将 Provider 的{{<hover label="offline" line="7">}}packagePullPolicy: Never{{</hover>}}.

```yaml {label="offline"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: offline-provider-aws
spec:
  package: provider-aws
  packagePullPolicy: Never
```

#### 安装选项

Providers 支持多种配置选项，可更改与安装相关的设置。

#### 提供商拉动政策

被引用 {{<hover label="pullpolicy" line="6">}}软件包拉取策略{{</hover>}}来定义 Crossplane 应何时将 Provider 软件包下载到本地 Crossplane 软件包缓存。

packagePullPolicy` 选项包括

* `IfNotPresent` - （**默认**）只下载不在缓存中的软件包。
* `Always` - 每分钟检查新软件包，并下载缓存中没有的匹配软件包。
* Never` - 永远不下载软件包。只从本地软件包缓存中安装软件包。

{{<hint "tip" >}}crossplane{{<hover label="pullpolicy" line="6">}}拉取策略{{</hover>}}的工作原理与 Kubernetes 容器镜像 [image pull policy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy)类似。

crossplane 支持像 Kubernetes 镜像一样使用标签和包摘要散列。{{< /hint >}}

例如，要 "始终 "下载给定的 Provider 软件包，可使用{{<hover label="pullpolicy" line="6">}}packagePullPolicy: Always{{</hover>}}配置。

```yaml {label="pullpolicy",copy-lines="6"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  packagePullPolicy: Always
# Removed for brevity
```

#### 修订激活政策

主动 "软件包 Revisions 是主动调节资源的软件包控制器。

默认情况下，Crossplane 会将最近安装的软件包修订版设置为 "激活"。

通过一个{{<hover label="revision" line="6">}}修订激活策略{{</hover>}}.

修订激活策略 {{<hover label="revision" line="6">}}修订激活策略{{</hover>}}选项有

* `Automatic` - （**默认**）自动激活最后安装的 Provider。
* `Manual` - 不自动激活 Provider。

例如，要将升级行为改为要求手动升级，可设置{{<hover label="revision" line="6">}}revisionActivationPolicy: Manual{{</hover>}}.

```yaml {label="revision"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  revisionActivationPolicy: Manual
# Removed for brevity
```

#### 包修订历史限制

当 Crossplane 安装同一 Provider 软件包的不同版本时，Crossplane 会创建一个新的 _revision_。

默认情况下，crossplane 维护一个 _Inactive_ Revisions。

{{<hint "note" >}}请阅读[Provider upgrade](#upgrade-a-provider)部分，了解更多关于使用软件包修订的信息。{{< /hint >}}

更改 Crossplane 通过 Provider 包维护的修订次数{{<hover label="revHistoryLimit" line="6">}}修订历史限制{{</hover>}}.

修订历史限制 {{<hover label="revHistoryLimit" line="6">}}修订历史限制{{</hover>}}字段是一个整数，默认值为 "1"。 通过设置{{<hover label="revHistoryLimit" line="6">}}修订历史限制{{</hover>}}设置为`0`。

例如，要更改默认设置并存储 10 个修订版本，请使用{{<hover label="revHistoryLimit" line="6">}}revisionHistoryLimit: 10{{</hover>}}.

```yaml {label="revHistoryLimit"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  revisionHistoryLimit: 10
# Removed for brevity
```

#### 从私人注册表安装 Provider

就像 Kubernetes 使用 `imagePullSecrets` 来 [从私有 registry 安装镜像](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)，Crossplane 也使用 `packagePullSecrets` 来从私有 registry 安装 Provider 软件包。

被引用 {{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}来提供一个 Kubernetes secret，以便在下载 Provider 软件包时用于身份验证。

{{<hint "important" >}}Kubernetes secret 必须与 crossplane 位于同一命名空间。{{</hint >}}

包 {{<hover label="pps" line="6">}}是一个secret列表。{{</hover>}}是一个secret列表。

例如，要使用名为{{<hover label="pps" line="6">}}example-secret 的secret{{</hover>}}配置一个{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}.

```yaml {label="pps"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  packagePullSecrets: 
    - name: example-secret
# Removed for brevity
```

{{<hint "note" >}}配置的 `packagePullSecrets` 不会传递给任何 Provider 软件包依赖项。{{< /hint >}}

#### 忽略依赖关系

默认情况下，Crossplane 会安装 Provider 软件包中列出的任何 [依赖项](#manage-dependencies)。

crossplane 可以通过以下方式忽略 Provider 软件包的依赖关系{{<hover label="pkgDep" line="6" >}}skipDependencyResolution{{</hover>}}.

例如，要禁用依赖关系解析，请配置{{<hover label="pkgDep" line="6" >}}skipDependencyResolution: true{{</hover>}}.

```yaml {label="pkgDep"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  skipDependencyResolution: true
# Removed for brevity
```

#### 忽略 crossplane 版本要求

在安装之前，Provider 软件包可能需要特定或最低 Crossplane 版本。 默认情况下，如果 Crossplane 版本不符合要求，Crossplane 不会安装 Provider。

crossplane 可以使用{{<hover label="xpVer" line="6">}}忽略 CrossplaneConstraints{{</hover>}}.

例如，要将 Provider 软件包安装到不支持的 crossplane 版本中，请配置{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints: true{{</hover>}}.

```yaml {label="xpVer"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  ignoreCrossplaneConstraints: true
# Removed for brevity
```

#### 管理依赖关系

Providers 软件包可能依赖于其他软件包，包括配置或其他 Providers。

如果 Crossplane 无法满足 Provider 软件包的依赖关系，Provider 会将 "HEALTHY "报告为 "False"。

例如，upbound AWS 参考平台的此安装为 `HEALTHY: false`。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-aws-s3 True False xpkg.upbound.io/upbound/provider-aws-s3:v0.41.0 12s
```

要查看有关 Provider 为何不是 "HEALTHY "的更多信息，请使用{{<hover label="depend" line="1">}}kubectl describe providerrevisions{{</hover>}}.

```yaml {copy-lines="1",label="depend"}
kubectl describe providerrevisions
Name:         provider-aws-s3-92206523fff4
API Version:  pkg.crossplane.io/v1
Kind:         ProviderRevision
Spec:
  Desired State:                  Active
  Image:                          xpkg.upbound.io/upbound/provider-aws-s3:v0.41.0
  Revision:                       1
Status:
  Conditions:
    Last Transition Time:  2023-10-10T21:06:39Z
    Reason:                UnhealthyPackageRevision
    Status:                False
    Type:                  Healthy
  Controller Ref:
    Name:
Events:
  Type Reason Age From Message
  ----     ------             ----               ----                                         -------
  Warning LintPackage 41s (x3 over 47s)  packages/providerrevision.pkg.crossplane.io incompatible Crossplane version: package is not compatible with Crossplane version (v1.10.0)
```

活动 {{<hover label="depend" line="17">}}活动{{</hover>}}显示{{<hover label="depend" line="20">}}警告{{</hover>}}并提示当前版本的 crossplane 不符合配置包要求。

## 升级一个提供商

要升级现有的 Provider，可通过应用新的 Provider 配置清单或使用 `kubectl edit providers` 编辑已安装的 Provider 包。

更新 Provider 的 `spec.package` 中的版本号并应用更改。 crossplane 安装新镜像并创建新的 `ProviderRevision` 。

ProviderRevision "允许 crossplane 存储已废弃的 Provider CRD，在您决定之前不会删除它们。

使用以下方法查看 `ProviderRevisions{{<hover label="getPR" line="1">}}kubectl get providerrevisions{{</hover>}}

```shell {label="getPR",copy-lines="1"}
kubectl get providerrevisions
NAME HEALTHY REVISION IMAGE STATE DEP-FOUND DEP-INSTALLED AGE
provider-aws-s3-dbc7f981d81f True 1 xpkg.upbound.io/upbound/provider-aws-s3:v0.37.0 Active 1 1 10d
provider-nop-552a394a8acc True 2 xpkg.upbound.io/crossplane-contrib/provider-nop:v0.3.0 Active 11d
provider-nop-7e62d2a1a709 True 1 xpkg.upbound.io/crossplane-contrib/provider-nop:v0.2.0 Inactive 13d
upbound-provider-family-aws-710d8cfe9f53 True 1 xpkg.upbound.io/upbound/provider-family-aws:v0.40.0 Active 10d
```

默认情况下，crossplane 只保留一个{{<hover label="getPR" line="5">}}非活动{{</hover>}}Provider.

请阅读 [revision history limit](#package-revision-history-limit) 部分以更改默认值。

一个 Provider 只能有一个修订版本{{<hover label="getPR" line="4">}}活动{{</hover>}}在同一时间内

## 删除一个提供商

使用 `kubectl delete provider` 删除 Provider 对象，从而移除 Provider。

{{< hint "warning" >}}删除 Provider 而不先删除 Provider 的托管资源可能会放弃资源。 外部资源不会被删除。

如果先删除 Provider，则必须通过云提供商手动删除外部资源。 必须通过移除其 Finalizer 手动删除托管资源。

有关删除废弃资源的更多信息，请阅读[crossplane 故障排除指南]({{<ref "/knowledge-base/guides/troubleshoot#deleting-when-a-resource-hangs" >}}).{{< /hint >}}

## 验证一个提供商

Providers 安装自己的 API，代表其支持的托管资源。 Providers 还可以创建部署、服务账户或 RBAC 配置。

使用以下功能查看 Provider 的状态

`kubectl get Provider` 获取 Provider

在安装过程中，Provider 将 `INSTALLED` 报告为 `True`，将 `HEALTHY` 报告为 `Unknown`。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
crossplane-contrib-provider-aws True Unknown xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0 63s
```

Provider 安装完成并准备就绪后，"HEALTHY "状态会被引用为 "True"。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
crossplane-contrib-provider-aws True True xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0 88s
```

{{<hint "important" >}}有些 Providers 会安装数百个 Kubernetes 自定义资源定义 (`CRD`)。 这可能会对规模不足的 API 服务器造成巨大压力，影响 Providers 的安装时间。

crossplane 社区有更多 [关于缩放 CRD 的详细信息](https://github.com/crossplane/crossplane/blob/master/design/one-pager-crd-scaling.md)。{{< /hint >}}

### 提供者条件

crossplane 为 Provider 引用了一套标准的 "条件"。 使用 "kubectl describe provider "可查看 Provider 的 "Status "下的条件。

```yaml
kubectl describe provider
Name:         my-provider
API Version:  pkg.crossplane.io/v1
Kind:         Provider
# Removed for brevity
Status:
  Conditions:
    Reason:      HealthyPackageRevision
    Status:      True
    Type:        Healthy
    Reason:      ActivePackageRevision
    Status:      True
    Type:        Installed
# Removed for brevity
```

#### 类型

Provider 的 "条件 "支持两种 "类型": 

* 类型: 已安装` - Provider 软件包已安装，但尚未准备好使用。
* 类型: Healthy` - Provider 软件包已可使用。

#### 原因

每个 "原因 "都与特定的 "类型 "和 "状态 "有关。 Crossplane 为 Provider "条件 "引用了以下 "原因"。

<!-- vale Google.Headings = NO -->

##### InactivePackageRevision

Reason: InactivePackageRevision"（原因: 非活动包修订版）表示该提供商包正在引用一个非活动的提供商包修订版。

<!-- vale Google.Headings = YES -->

```yaml
Type: Installed
Status: False
Reason: InactivePackageRevision
```

<!-- vale Google.Headings = NO -->

##### ActivePackageRevision

<!-- vale Google.Headings = YES -->

Provider 软件包是当前的软件包修订版，但 crossplane 尚未完成软件包修订版的安装。

{{< hint "tip" >}}陷入这种状态的提供商是因为数据包修订（Package Revisions）出了问题。

更多详情请被引用 `kubectl describe providerrevisions`。{{< /hint >}}

```yaml
Type: Installed
Status: True
Reason: ActivePackageRevision
```

<!-- vale Google.Headings = NO -->

##### HealthyPackageRevision

Provider 已完全安装完毕，随时可以使用。

{{<hint "tip" >}}原因: HealthyPackageRevision "是工作中的 Provider 的正常状态。{{< /hint >}}

<!-- vale Google.Headings = YES -->

```yaml
Type: Healthy
Status: True
Reason: HealthyPackageRevision
```

<!-- vale Google.Headings = NO -->

##### UnhealthyPackageRevision

<!-- vale Google.Headings = YES -->

安装 Provider 软件包修订版时出现错误，导致 crossplane 无法安装 Provider 软件包。

{{<hint "tip" >}}使用 `kubectl describe providerrevisions` 获取更多有关包修订失败原因的详细信息。{{< /hint >}}

```yaml
Type: Healthy
Status: False
Reason: UnhealthyPackageRevision
```

<!-- vale Google.Headings = NO -->

##### UnknownPackageRevisionHealth

<!-- vale Google.Headings = YES -->

Provider 包修订版的状态是 "未知"。 Provider 包修订版可能正在安装或有问题。

{{<hint "tip" >}}使用 `kubectl describe providerrevisions` 获取更多有关包修订失败原因的详细信息。{{< /hint >}}

```yaml
Type: Healthy
Status: Unknown
Reason: UnknownPackageRevisionHealth
```

## 配置 Provider

Providers 有两种不同类型的配置: 

* 控制器配置_可更改在 Kubernetes 集群内运行的 Provider pod 的设置。
的设置。例如，在
例如，在 Provider pod 上设置 "toleration"。
* 提供程序配置_可更改与外部提供程序通信时被引用的设置。
与外部提供程序通信时使用的设置。例如，云提供商身份验证。

{{<hint "important" >}}将 `ControllerConfig` 对象应用到 Provider。

将 `ProviderConfig` 对象应用到 managed 资源。{{< /hint >}}

### 控制器配置

{{< hint "important" >}}

<!-- vale write-good.Passive = NO -->

<!-- vale gitlab.FutureTense = NO -->

在 v1.11 中，"ControllerConfig "类型已被弃用，并将在以后的发布中移除。

<!-- vale write-good.Passive = YES -->

<!-- vale gitlab.FutureTense = YES -->

[部署运行时配置]({{<ref "#runtime-configuration" >}}) 可替代控制器配置，在 v1.14+ 中可用。{{< /hint >}}

对 Provider 应用 crossplane `ControllerConfig` 会更改 Provider pod 的设置。[Crossplane ControllerConfig schema](https://doc.crds.dev/github.com/crossplane/crossplane/pkg.crossplane.io/ControllerConfig/v1alpha1) 定义了受支持的 ControllerConfig 设置集。

ControllerConfigs 最常见的用例是为 Provider 的 pod 启用可选服务提供 "参数"。 例如，为 Provider 启用[外部存储](https://docs.crossplane.io/knowledge-base/integrations/vault-as-secret-store/#enable-external-secret-stores-in-the-provider)。

每个 Provider 都会确定其支持的 "参数 "集。

### 运行时配置

{{<hint "important" >}}DeploymentRuntimeConfigs "是测试版功能。

默认情况下它是开启的，你可以通过在 crossplane 部署中传递 `--enable-deployment-runtime-configs=false`来禁用它。{{< /hint >}}

运行时配置是一种通用机制，用于配置具有运行时的 crossplane 包（即 `Provider` 和 `Functions`）的运行时。 它取代了已废弃的 `ControllerConfig` 类型，在 v1.14+ 中可用。

在默认配置下，crossplane 使用 Kubernetes 部署来为软件包部署运行时，更具体地说，是为 "Provider "部署控制器，或为 "Function "部署 gRPC 服务器。 可以通过被引用 "DeploymentRuntimeConfig "并在 "Provider "或 "Function "对象中引用它来配置运行时清单。

{{<hint "note" >}}与 `ControllerConfig` 不同，`DeploymentRuntimeConfig` 嵌入了整个 Kubernetes 部署规范，可以更灵活地配置运行时。详情请参考[设计文档](https://github.com/crossplane/crossplane/blob/2c5e7f07ba9e3d83d1c85169bbde685de8514ab8/design/one-pager-package-runtime-config.md)。{{< /hint >}}

举例来说，如果要通过在控制器中添加 `--enable-external-secret-stores`参数来启用 `Provider` 的外部secret存储 alpha 功能，可以应用下面的方法: 

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-iam
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-iam:v0.37.0
  runtimeConfigRef:
    name: enable-ess
---
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: enable-ess
spec:
  deploymentTemplate:
    spec:
      selector: {}
      template:
        spec:
          containers:
            - name: package-runtime
              args:
                - --enable-external-secret-stores
```

请注意，软件包管理器使用 `package-runtime` 作为运行时容器的名称。 当你使用不同的容器名称时，软件包管理器会将其作为侧载容器引入，而不是修改软件包运行时容器。

<!-- vale write-good.Passive = NO -->

packages 管理器对某些字段有意见，以确保

<!-- vale write-good.Passive = YES -->

例如，如果未设置复制计数，它会将其默认为 1，并覆盖标签选择器，以确保部署和服务相匹配。 它还会注入任何必要的环境变量、端口以及卷和卷挂载。

Provider "或 "Functions "的 "spec.runtimeConfigRef.name "字段默认值为 "default"，这意味着如果未指定，Crossplane 将使用默认运行时配置。 Crossplane 可确保始终有一个默认运行时配置。

<!-- vale gitlab.FutureTense = NO -->

但如果配置已经存在，则不会更改。

<!-- vale gitlab.FutureTense = YES -->

允许用户根据自己的需要定制默认运行时配置。

{{<hint "tip" >}}

<!-- vale gitlab.SubstitutionWarning = NO -->

由于 `DeploymentRuntimeConfig` 与 Kubernetes `Deployment` 使用相同的模式

<!-- vale gitlab.SubstitutionWarning = YES -->

例如，如果只想更改 `replicas` 字段，则需要传递以下内容: 

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: multi-replicas
spec:
  deploymentTemplate:
    spec:
      replicas: 2
      selector: {}
      template: {}
```

{{< /hint >}}

#### 配置运行时部署规范

软件包管理器会以 `DeploymentRuntimeConfig` 中提供的部署规范为基础，按照以下规则为软件包运行时构建部署规范: 

* 注入软件包运行时容器，作为 `containers` 数组中的第一个容器，名称为 `package-runtime`。
* 如果未 Provider，则默认使用以下内容: 
    - `spec.replicas` 为 1。
    - 镜像拉取策略为 `IfNotPresent`。
    - Pod 安全上下文为
        yaml
        runAsNonRoot: true
        runAsUser: 2000
        runAsGroup: 2000
        ```
    - 运行时容器的安全上下文为
        ```yaml
        allowPrivilegeEscalation: false
        privileged: false
        runAsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
        ```
* 应用以下内容: 
    - **设置*** `metadata.namespace` 为 crossplane 名称空间。
    - **Sets** `metadata.ownerReferences` such that the deployment owned by the packages revision.
    - **设置*** `spec.selectors` 使用生成的标签。
    - **使用创建的**服务账户**设置**`spec.serviceAccount`。
    - **添加**Package 规范中提供的拉取secret作为镜像拉取secret，即 `spec.packagePullSecrets`。
    - **使用软件包规格中提供的值设置**镜像拉取策略**，`spec.packagePullPolicy`。
    - **向运行时容器添加**必要的**端口。
    - **将必要的**端口**添加到运行时容器。
    - 通过在运行时容器中添加***必要的**卷**、**卷挂载**和**环境**来挂载 TLS secrets。

#### 配置运行时资源的元数据

DeploymentRuntimeConfig "还能配置运行时资源的以下元数据，即 "部署"、"服务帐户 "和 "服务": 

* 名称
* 标签
* Annotations

下面的示例显示了如何配置服务帐户的名称和部署的标签: 

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: my-runtime-config
spec:
  deploymentTemplate:
    metadata:
      labels:
        my-label: my-value
  serviceAccountTemplate:
    metadata:
      name: my-service-account
```

### Provider 配置

ProviderConfig "决定了 Provider 与外部提供程序通信时使用的设置。 每个提供程序都决定了其 "ProviderConfig "的可用设置。

<!-- vale write-good.Weasel = NO -->

<!-- allow "usually" -->

Provider 身份验证通常使用 "ProviderConfig "进行配置。 例如，要在 Provider AWS 中使用基本密钥对身份验证，就需要使用一个{{<hover label="providerconfig" line="2" >}}提供商配置{{</hover >}}{{<hover label="providerconfig" line="5" >}}规格{{</hover >}}定义了{{<hover label="providerconfig" line="6" >}}凭据{{</hover >}}以及 Provider pod 应在 Kubernetes{{<hover label="providerconfig" line="7" >}}Secrets{{</hover >}}对象中，并使用名为{{<hover label="providerconfig" line="10" >}}aws-creds{{</hover >}}.

<!-- vale write-good.Weasel = YES -->

```yaml {label="providerconfig"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

{{< hint "important" >}}不同 Provider 的身份验证配置可能不同。

请阅读特定 Provider 的文档，了解为该 Provider 配置身份验证的说明。{{< /hint >}}

<!-- vale write-good.TooWordy = NO -->

<!-- allow multiple -->

ProviderConfig 对象适用于单个托管资源。 单个 Provider 可以通过 ProviderConfigs 验证多个用户或账户。

<!-- vale write-good.TooWordy = YES -->

每个账户的凭据都与唯一的 ProviderConfig 相关联。 创建托管资源时，附加所需的 ProviderConfig。

例如，两个名为{{<hover label="user" line="4">}}用户密钥{{</hover >}}和{{<hover label="admin" line="4">}}admin-keys{{</hover >}}被引用了不同的 Kubernetes secrets。

```yaml {label="user"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: user-keys
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: my-key
      key: secret-key
```

```yaml {label="admin"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: admin-keys
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: admin-key
      key: admin-secret-key
```

创建托管资源时应用 ProviderConfig。

这会创建一个 AWS {{<hover label="user-bucket" line="2" >}}桶{{< /hover >}}资源。{{<hover label="user-bucket" line="9" >}}用户密钥{{< /hover >}}ProviderConfig.

```yaml {label="user-bucket"}
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: user-bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: user-keys
```

这会创建第二个 {{<hover label="admin-bucket" line="2" >}}桶{{< /hover >}}资源，使用被引用的{{<hover label="admin-bucket" line="9" >}}admin-keys{{< /hover >}}ProviderConfig.

```yaml {label="admin-bucket"}
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: user-bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: admin-keys
```