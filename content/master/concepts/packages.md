---

title: 配置包
描述: "包将多个 crossplane 资源组composition一个可移植的 OCI 镜像"
altTitle: "crossplane软件包"
weight: 200

---

一个 _Configuration_ 包是一个[OCI 容器镜像](https://opencontainers.org/)，包含[Composition]({{<ref "./compositions" >}})、[复合资源定义]({{<ref "./composite-resource-definitions" >}}) 和任何所需的 [Provider]({{<ref "./providers">}}) 或 [函数]({{<ref "./composition-functions" >}}).

配置包使你的 crossplane 配置完全可移植。

{{<hint "important" >}}crossplane [Providers]({{<ref "./providers">}}) 和 [Functions]({{<ref "./composition-functions">}}) 也是 crossplane 软件包。

本文件介绍如何安装和管理配置包。

请参阅 [Provider]({{<ref "./providers">}}) 和 [Composition Functions]({{<ref "./composition-functions">}}) 章节，详细了解它们对 packages 的用法。{{< /hint >}}

## 安装配置

安装带 crossplane 的配置{{<hover line="2" label="install">}}配置{{</hover>}}对象安装配置 {{<hover line="6" label="install">}}spec.packages{{</hover>}}值设置为配置包的位置。

例如安装 [upbound AWS 参考平台](https://marketplace.upbound.io/configurations/upbound/platform-ref-aws/v0.6.0)、

```yaml {label="install"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  package: xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0
```

crossplane 会安装配置中列出的 Composition、复合资源定义和 Providers。

### 用 Helm 安装

Crossplane 支持在初始安装 Crossplane 时使用 Crossplane Helm 图表安装配置。

被引用{{<hover label="helm" line="5" >}}--set configuration.packages{{</hover >}}参数与 `helm install` 一起使用。

例如，安装 upbound AWS 参考平台、

```shell {label="helm"}
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--set configuration.packages='{xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0}'
```

#### 离线安装

crossplane 从本地软件包缓存中安装软件包。默认情况下，crossplane 软件包缓存是一个 [emptyDir volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)。

配置 Crossplane，使其被引用[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，以使用包含配置镜像的存储位置。阅读[Crossplane 安装文档]( )中有关配置 Crossplane Pod 设置的更多信息。{{<ref "../software/install#customize-the-crossplane-helm-chart">}}).

Provider 配置的 `.xpkg` 文件名并设置{{<hover label="offline" line="7">}}packagePullPolicy: Never{{</hover>}}.

例如，要安装本地存储的 upbound AWS 参考平台版本，请设置{{<hover label="offline" line="6">}}package{{</hover>}}为本地文件名，并将配置的{{<hover label="offline" line="7">}}packagePullPolicy: Never{{</hover>}}.

```yaml {label="offline"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: offline-platform-ref-aws
spec:
  package: platform-ref-aws
  packagePullPolicy: Never
```

#### 安装选项

配置支持多个选项来更改配置包相关设置。

#### 配置修订

当安装现有配置的更新版本时，crossplane 会创建一个新的配置修订版。

使用以下命令查看配置修订版{{<hover label="rev" line="1">}}kubectl get configurationrevisions{{</hover>}}.

```shell {label="rev",copy-lines="1"}
kubectl get configurationrevisions
NAME HEALTHY REVISION IMAGE STATE DEP-FOUND DEP-INSTALLED AGE
platform-ref-aws-1735d56cd88d True 2 xpkg.upbound.io/upbound/platform-ref-aws:v0.5.0 Active 2 2 46s
platform-ref-aws-3ac761211893 True 1 xpkg.upbound.io/upbound/platform-ref-aws:v0.4.1 Inactive 5m13s
```

每次只有一个修订处于活动状态，活动修订决定可用资源，包括 Composition 和复合资源定义。

默认情况下，crossplane 只保留一个 _Inactive_ Revisions。

更改 Crossplane 通过配置包维护的修订次数{{<hover label="revHistory" line="6">}}修订历史限制{{</hover>}}.

修订历史限制 {{<hover label="revHistory" line="6">}}修订历史限制{{</hover>}}字段是一个整数，默认值为 "1"。 通过设置{{<hover label="revHistory" line="6">}}修订历史限制{{</hover>}}设置为`0`。

例如，要更改默认设置并存储 10 个修订版本，请使用{{<hover label="revHistory" line="6">}}revisionHistoryLimit: 10{{</hover>}}.

```yaml {label="revHistory"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  revisionHistoryLimit: 10
# Removed for brevity
```

#### 配置包拉取策略

被引用 {{<hover label="pullpolicy" line="6">}}packagePullPolicy{{</hover>}}来定义 Crossplane 应在何时将配置包下载到本地的 Crossplane 包缓存。

packagePullPolicy` 选项包括

* `IfNotPresent` - （**默认**）只下载不在缓存中的软件包。
* `Always` - 每分钟检查新软件包，并下载缓存中没有的匹配软件包。
* Never` - 永远不下载软件包。只从本地软件包缓存中安装软件包。

{{<hint "tip" >}}crossplane{{<hover label="pullpolicy" line="6">}}拉取策略{{</hover>}}的工作原理与 Kubernetes 容器镜像 [image pull policy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy)类似。

crossplane 支持像 Kubernetes 镜像一样使用标签和包摘要散列。{{< /hint >}}

例如，要 "始终 "下载给定的配置软件包，请使用{{<hover label="pullpolicy" line="6">}}packagePullPolicy: Always{{</hover>}}配置。

```yaml {label="pullpolicy",copy-lines="6"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  packagePullPolicy: Always
# Removed for brevity
```

#### 修订激活政策

主动 "软件包 Revisions 是主动调节资源的软件包控制器。

默认情况下，Crossplane 会将最近安装的软件包修订版设置为 "激活"。

使用{{<hover label="revision" line="6">}}修订激活策略{{</hover>}}.

修订激活策略 {{<hover label="revision" line="6">}}修订激活策略{{</hover>}}选项有

* 自动"-（**默认**）自动激活上次安装的配置。
* `Manual` - 不自动激活配置。

例如，要将升级行为改为要求手动升级，可设置{{<hover label="revision" line="6">}}revisionActivationPolicy: Manual{{</hover>}}.

```yaml {label="revision"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  revisionActivationPolicy: Manual
# Removed for brevity
```

#### 从私人注册表安装配置

就像 Kubernetes 使用 "imagePullSecrets "来[从私有 registry 安装镜像](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)一样，Crossplane 也使用 "packagePullSecrets "来从私有 registry 安装配置包。

被引用 {{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}来提供一个 Kubernetes secrets，以便在下载配置包时用于身份验证。

{{<hint "important" >}}Kubernetes secret 必须与 crossplane 位于同一命名空间。{{</hint >}}

包 {{<hover label="pps" line="6">}}是一个secret列表。{{</hover>}}是一个secret列表。

例如，要使用名为{{<hover label="pps" line="6">}}example-secret 的secret{{</hover>}}配置一个{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}.

```yaml {label="pps"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  packagePullSecrets: 
    - name: example-secret
# Removed for brevity
```

#### 忽略依赖关系

默认情况下，crossplane 会安装配置包中列出的任何 [依赖项](#manage-dependencies)。

crossplane 可以忽略配置包的依赖关系，使用{{<hover label="pkgDep" line="6" >}}skipDependencyResolution{{</hover>}}.

{{< hint "warning" >}}大多数配置包括所需 Providers 的依赖关系。

如果 "配置 "忽略了依赖关系，则必须手动安装所需的 Provider。{{< /hint >}}

例如，要禁用依赖关系解析，请配置{{<hover label="pkgDep" line="6" >}}skipDependencyResolution: true{{</hover>}}.

```yaml {label="pkgDep"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  skipDependencyResolution: true
# Removed for brevity
```

#### 忽略 crossplane 版本要求

默认情况下，如果 Crossplane 版本不符合要求，Crossplane 不会安装配置包。

crossplane 可以使用{{<hover label="xpVer" line="6">}}忽略 CrossplaneConstraints{{</hover>}}.

例如，要将配置包安装到不支持的 crossplane 版本中，请配置{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints: true{{</hover>}}.

```yaml {label="xpVer"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  ignoreCrossplaneConstraints: true
# Removed for brevity
```

### 验证配置

使用{{<hover label="verify" line="1">}}获取配置{{</hover >}}.

工作配置将 "已安装 "和 "健康 "报告为 "真"。

```shell {label="verify",copy-lines="1"}
kubectl get configuration
NAME INSTALLED HEALTHY PACKAGE AGE
platform-ref-aws True True xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0 54s
```

#### 管理依赖关系

配置包可能包括对其他包（包括功能、Provider 或其他配置）的依赖。

如果 crossplane 无法满足 "配置 "的依赖关系，则 "配置 "会将 "HEALTHY "报告为 "False"。

例如，upbound AWS 参考平台的此安装为 `HEALTHY: false`。

```shell {copy-lines="1"}
kubectl get configuration
NAME INSTALLED HEALTHY PACKAGE AGE
platform-ref-aws True False xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0 71s
```

要查看更多关于配置为何不 "健康 "的信息，请引用{{<hover label="depend" line="1">}}kubectl describe configurationrevisions{{</hover>}}.

```yaml {copy-lines="1",label="depend"}
kubectl describe configurationrevision
Name:         platform-ref-aws-a30ad655c769
API Version:  pkg.crossplane.io/v1
Kind:         ConfigurationRevision
# Removed for brevity
Spec:
  Desired State:                  Active
  Image:                          xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0
  Revision:                       1
Status:
  Conditions:
    Last Transition Time:  2023-10-06T20:08:14Z
    Reason:                UnhealthyPackageRevision
    Status:                False
    Type:                  Healthy
  Controller Ref:
    Name:
Events:
  Type Reason Age From Message
  ----     ------       ----               ----                                              -------
  Warning LintPackage 29s (x2 over 29s)  packages/configurationrevision.pkg.crossplane.io incompatible Crossplane version: package is not compatible with Crossplane version (v1.12.0)
```

活动 {{<hover label="depend" line="18">}}活动{{</hover>}}显示{{<hover label="depend" line="21">}}警告{{</hover>}}并提示当前版本的 crossplane 不符合配置包要求。

## 创建配置

crossplane 配置包是包含一个或多个 YAML 文件的 [OCI 容器镜像](https://opencontainers.org/)。

{{<hint "important" >}}配置包完全符合 OCI 标准，任何能生成 OCI 镜像的工具都能生成配置包。

强烈建议使用 Crossplane 命令行工具为 Crossplane 软件包的构建提供错误检查和格式化。

使用第三方工具构建软件包时，请阅读[Crossplane 软件包规范](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md) 了解软件包要求。{{</hint >}}

配置包需要一个 `crossplane.yaml` 文件，并可能包括 Composition 和 CompositeResourceDefinition 文件。

<!-- vale Google.Headings = NO -->

### crossplane.yaml 文件

<!-- vale Google.Headings = YES -->

要使用 crossplane CLI 构建配置包，请创建一个名为{{<hover label="cfgMeta" line="1">}}crossplane.yaml 的文件。{{</hover>}}文件。{{<hover label="cfgMeta" line="1">}}crossplane.yaml{{</hover>}}文件定义了配置的要求和名称。

{{<hint "important" >}}crossplane CLI 只支持名为 `crossplane.yaml` 的文件。{{< /hint >}}

配置包被引用为{{<hover label="cfgMeta" line="2">}}meta.pkg.crossplane.io{{</hover>}}crossplane API group。

中指定任何其他配置、功能或 Provider。{{<hover label="cfgMeta" line="7">}}依赖于{{</hover>}}可选择使用{{<hover label="cfgMeta" line="9">}}版本{{</hover>}}选项来要求特定或最小的软件包版本。

您还可以使用{{<hover label="cfgMeta" line="11">}}crossplane.version{{</hover>}}选项为该配置定义特定或最低版本。

{{<hint "note" >}}定义 {{<hover label="cfgMeta" line="10">}}crossplane{{</hover>}}对象或所需版本是可选的。{{< /hint >}}

```yaml {label="cfgMeta",copy-lines="all"}
$ cat crossplane.yaml
apiVersion: meta.pkg.crossplane.io/v1alpha1
kind: Configuration
metadata:
  name: test-configuration
spec:
  dependsOn:
    - provider: xpkg.upbound.io/crossplane-contrib/provider-aws
      version: ">=v0.36.0"
  crossplane:
    version: ">=v1.12.1-0"
```

#### 构建 package

被引用[crossplane CLI]({{<ref "../cli">}}) 命令 `crossplane xpkg build --package-root=<directory>` 创建软件包。

其中 `<directory>` 是包含 `crossplane.yaml` 文件和任何 Composition 或 CompositeResourceDefinition YAML 文件的目录。

CLI 会递归搜索目录中的 `.yml` 或 `.yaml` 文件，以便将其包含在 packages 中。

{{<hint "important" >}}您必须忽略任何其他带有 `--ignore=<file_list>` 的 YAML 文件。例如，`crossplane xpkg build --package-root=test-directory --ignore=".tmp/*"`.

不支持包括claim在内的非 Composition 或 CompositeResourceDefinitions 的 YAML 文件。{{</hint >}}

默认情况下，crossplane 会创建一个配置名称和软件包内容 SHA-256 哈希值的 `.xpkg` 文件。

例如 {{<hover label="xpkgName" line="2">}}配置{{</hover>}}名为 {{<hover label="xpkgName" line="4">}}的配置。{{</hover>}}crossplane CLI 会构建一个名为 `test-configuration-e8c244f6bf21.xpkg` 的软件包。

```yaml {label="xpkgName"}
apiVersion: meta.pkg.crossplane.io/v1alpha1
kind: Configuration
metadata:
  name: test-configuration
# Removed for brevity
```

使用 `--output=<filename>.xpkg` 选项指定输出文件。

例如，要从名为 `test-directory` 的目录中构建软件包，并在当前工作目录中生成名为 `test-package.xpkg` 的软件包，请使用以下命令: 

```shell
crossplane xpkg build --package-root=test-directory --output=test-package.xpkg
```

```shell
ls -1 ./
test-directory
test-package.xpkg
```
