---

title: 安装 crossplane
weight: 100

---

Crossplane 安装到现有的 Kubernetes 集群中，创建 `Crossplane` pod，实现 Crossplane _Provider_ 资源的安装。

{{< hint type="tip" >}}如果没有 Kubernetes 集群，请使用 [Kind](https://kind.sigs.k8s.io/)在本地创建一个。{{< /hint >}}

## 先决条件

* 主动[支持的 Kubernetes 版本](https://kubernetes.io/releases/patch-releases/#support-period)
* [Helm](https://helm.sh/docs/intro/install/)版本`v3.2.0`或更高版本

## 安装 crossplane

使用 Crossplane 出版的_Helm 图表_安装 Crossplane。

### 添加 crossplane helm 存储库

使用 `helm repo add` 命令添加 crossplane 软件仓库。

```shell
helm repo add crossplane-stable https://charts.crossplane.io/stable
```

使用 `helm repo update` 更新本地 Helm 图表缓存。

```shell
helm repo update
```

### 安装 crossplane helm 图表

使用 `helm install` 安装 crossplane 舵图。

{{< hint "tip" >}}使用 `helm install --dry-run --debug` 选项查看 Crossplane 对集群所做的更改。 Helm 会显示它在不对 Kubernetes 集群做更改的情况下应用了哪些配置。{{< /hint >}}

crossplane 创建并安装到 `crossplane-system` 命名空间。

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane
```

使用 `kubectl get pods -n crossplane-system` 查看已安装的 crossplane pod。

```shell {copy-lines="1"}
kubectl get pods -n crossplane-system
NAME READY STATUS RESTARTS AGE
crossplane-6d67f8cd9d-g2gjw 1/1 Running 0 26m
crossplane-rbac-manager-86d9b5cf9f-2vc4s 1/1 Running 0 26m
```

{{< hint "tip" >}}使用"--版本<version>"选项安装特定版本的 crossplane。 例如，安装版本 "1.10.0": 

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane \
--version 1.10.0
```

{{< /hint >}}

## 已安装的部署

Crossplane 在 `crossplane-system` 名称空间中创建了两个 Kubernetes _deployments_，用于部署 Crossplane pod。

```shell {copy-lines="1"}
kubectl get deployments -n crossplane-system
NAME READY UP-TO-DATE AVAILABLE AGE
crossplane 1/1 1 1 8m13s
crossplane-rbac-manager 1/1 1 1 8m13s
```

### Crossplane 部署

Crossplane 部署从 "crossplane-init 容器 "开始，"init "容器会将 Crossplane_自定义资源定义_安装到 Kubernetes 集群中。

在`init`容器完成后，`crossplane` pod 会管理两个 Kubernetes 控制器。

* 软件包管理器控制器会安装

Provider 和配置包。

* 组件控制器安装并管理

crossplane _Composite Resource Definitions_、_Compositions_ 和_Claims_。

### crossplane RBAC 管理器的部署

crossplane-rbac-manager "会为已安装的 crossplane _Provider_ 及其 _Custom Resource Definitions_ 创建和管理 Kubernetes _ClusterRoles_。

Crossplane RBAC 管理器设计文档](https://github.com/crossplane/crossplane/blob/master/design/design-doc-rbac-manager.md) 中有更多关于已安装的 _ClusterRoles_ 的信息。

## 安装选项

### 定制 crossplane 舵图

Crossplane 支持在安装时通过配置 Helm 图表进行定制。

通过命令行或 Helm _values_ 文件应用自定义功能。

<!-- vale gitlab.Substitutions = NO -->

<!-- allow lowercase yaml -->

{{<expand "All Crossplane customization options" >}}{{< table "table table-hover table-striped table-sm">}}| Parameter | Description | Default |
| --- | --- | --- |
| `affinity` | Add `affinities` to the Crossplane pod deployment. | `{}` |
| `args` | Add custom arguments to the Crossplane pod. | `[]` |
| `configuration.packages` | A list of Configuration packages to install. | `[]` |
| `customAnnotations` | Add custom `annotations` to the Crossplane pod deployment. | `{}` |
| `customLabels` | Add custom `labels` to the Crossplane pod deployment. | `{}` |
| `deploymentStrategy` | The deployment strategy for the Crossplane and RBAC Manager pods. | `"RollingUpdate"` |
| `extraEnvVarsCrossplane` | Add custom environmental variables to the Crossplane pod deployment. Replaces any `.` in a variable name with `_`. For example, `SAMPLE.KEY=value1` becomes `SAMPLE_KEY=value1`. | `{}` |
| `extraEnvVarsRBACManager` | Add custom environmental variables to the RBAC Manager pod deployment. Replaces any `.` in a variable name with `_`. For example, `SAMPLE.KEY=value1` becomes `SAMPLE_KEY=value1`. | `{}` |
| `extraObjects` | To ad{{< /table >}}{{< /expand >}}

<!-- vale gitlab.Substitutions = YES -->

#### 命令行定制

使用 `helm install crossplane --set<setting>=<value>` 命令行应用自定义设置。

例如，更改镜像拉动策略: 

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
--set image.pullPolicy=Always
```

Helm 支持以逗号分隔的参数。

例如，更改镜像拉取策略和副本数量: 

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
--set image.pullPolicy=Always,replicas=2
```

#### Helm Values 文件

使用 `helm install crossplane -f<filename>` 应用 Helm _values_ 文件中的自定义设置。

YAML 文件定义了自定义设置。

例如，更改镜像拉取策略和副本数量: 

创建包含自定义设置的 YAML。

```yaml
replicas: 2

image:
  pullPolicy: Always
```

用 `helm install` 应用该文件: 

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
-f settings.yaml
```

#### 功能标志

Crossplane 通过特性标志引入新特性。 默认情况下，alpha 特性是关闭的。 Crossplane 默认启用 beta 特性。 要启用特性标志，请在 helm chart 中设置 `args` 值。 可用的特性标志可以通过运行 `crossplane core start --help` 直接找到，也可以查看下表。

{{< expand "Feature flags" >}}{{< table caption="Feature flags" >}}| Status | Flag | Description |
| --- | --- | --- |
| Beta | `--enable-composition-revisions` | Enable support for CompositionRevisions. |
| Beta | `--enable-composition-webhook-schema-validation` | Enable Composition validation using schemas. |
| Alpha | `--enable-composition-functions` | Enable support for Composition Functions. |
| Alpha | `--enable-environment-configs` | Enable support for EnvironmentConfigs. |
| Alpha | `--enable-external-secret-stores` | Enable support for External Secret Stores. |
| Alpha | `--enable-usages` | Enable support for Usages. |
| Alpha | `--enable-realtime-compositions` | Enable support for real time compositions. |{{< /table >}}{{< /expand >}}

在 `values.yaml` 文件中或安装时使用 `--set` flag 设置这些 flag，例如:  `--set args='{"--enable-composition-functions","--enable-composition-webhook-schema-validation"}'`.

### 安装发布前的 crossplane 版本

从 "master "Crossplane Helm 频道安装预发布版本的 Crossplane。

master "频道中的版本正在开发中，可能不稳定。

{{< hint "warning" >}}不要在生产中使用 crossplane `master` 发布。 只能使用 `stable` 通道。 只能将 `master` 用于测试和开发。{{< /hint >}}

#### 添加 crossplane 主 Helm 资源库

使用 `helm repo add` 命令添加 crossplane 软件仓库。

```shell
helm repo add crossplane-master https://charts.crossplane.io/master/
```

使用 `helm repo update` 更新本地 Helm 图表缓存。

```shell
helm repo update
```

#### 安装 crossplane 主 helm 图表

用 `helm install` 安装 crossplane `master` 舵图。

{{< hint "tip" >}}使用 `helm install --dry-run --debug` 选项查看 Crossplane 对集群所做的更改。 Helm 会显示它在不对 Kubernetes 集群做更改的情况下应用了哪些配置。{{< /hint >}}

crossplane 创建并安装到 `crossplane-system` 命名空间。

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-master/crossplane \
--devel
```

## crossplane 分布情况

第三方供应商可能会维护自己的 crossplane 发行版。 供应商支持的发行版可能具有社区 Crossplane 发行版中没有的功能或工具。

CNCF 认证第三方发行版为"[符合](https://github.com/cncf/crossplane-conformance) "Community Crossplane 发行版。

###供应商

以下是提供符合要求的 crossplane 发行版的供应商。

#### Upbound

Upbound是crossplane的创始人，它维护着一个名为[Universal Crossplane](https://www.upbound.io/products/universal-crossplane)（`UXP`）的免费开源crossplane发行版。

在 [upbound UXP 文档](https://docs.upbound.io/uxp/install/) 中查找有关 UXP 的信息。