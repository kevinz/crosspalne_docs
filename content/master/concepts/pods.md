---

title: crossplane pod
weight: 1
说明: Crossplane 安装的组件及其功能的背景介绍。

---

基本的 crossplane 安装由两个 pod 组成: "crossplane "pod 和 "crossplane-rbac-manager "pod。 这两个 pod 默认安装在 "crossplane-system "命名空间中。

## Crossplane pod

### Init container

在启动核心 crossplane 容器之前，会运行一个 _init_ 容器。init 容器会安装核心 crossplane [自定义资源定义](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) (`CRDs`)，配置 crossplane webhook，并安装提供的 Provider 或配置。

{{<hint "tip" >}}Kubernetes 文档包含有关 [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) 的更多信息。{{< /hint >}}

init 容器设置的设置包括用 crossplane 安装 Provider 或 Configuration 包、自定义 crossplane 安装的 namespace 以及定义 webhook 配置。

init 容器安装的核心 CRD 包括

* CompositeResourceDefinitions、Composition、Configurations 和 Providers
* 用于管理软件包依赖关系的锁
* DeploymentRuntimeConfigs 用于将设置应用于已安装的 Provider 和函数
* StoreConfigs 用于连接外部秘密存储，例如

[HashiCorp Vault](https://www.vaultproject.io/)

{{< hint "note" >}}

安装 Crossplane]({{< ref "../software/install" >}}) 部分有更多关于自定义安装 crossplane 的信息。{{< /hint >}}

crossplane pod 上的状态 "Init "表示 init 容器正在运行。

```shell
kubectl get pods -n crossplane-system
NAME READY STATUS RESTARTS AGE
crossplane-9f6d5cd7b-r9j8w 0/1 Init:0/1 0 6s
```

init 容器会自动完成并启动 crossplane 核心容器。

```shell
kubectl get pods -n crossplane-system
NAME READY STATUS RESTARTS AGE
crossplane-9f6d5cd7b-r9j8w 1/1 Running 0 15s
```

#### 核心容器

主 crossplane 容器被称为 _core_ 容器，它负责强制执行 crossplane 资源的理想状态、管理领导者选举和处理 webhook。

{{<hint "note" >}}Crossplane pod 只调节 Crossplane 核心组件，包括 claims 和 Composite 资源。 Provider 负责调节其管理的资源。{{< /hint >}}

#### 调节循环

核心容器在一个_reconcile循环中运行，不断检查已部署资源的状态，并纠正任何 "漂移"。 在检查完一个资源后，crossplane 会等待一段时间，然后再次检查。

crossplane 通过 Kubernetes [_watch_](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)或定期轮询来监控资源。有些资源可能同时被监视和轮询。

Crossplane 请求 API 服务器通知 Crossplane 对象上的任何变化。 这种通知工具是_watch_。

监视对象包括 Provider、托管资源和 CompositeResourceDefinitions。

对于 Kubernetes 无法提供监控的对象，crossplane 会定期轮询资源以了解其状态。 默认轮询率为一分钟。 使用 pod 参数"--poll-interval "更改轮询率。

降低轮询间隔值会导致 crossplane 更频繁地轮询资源。 这会增加 crossplane pod 的负载，导致更频繁地调用 Provider API。

<!-- vale write-good.TooWordy = NO -->

<!-- allow "maximum" -->

增加轮询间隔会降低 crossplane 轮询资源的频率，从而延长 Crossplane 发现云提供商中需要更新的变更之前的最长时间。

<!-- vale write-good.TooWordy = YES -->

managed resources 使用轮询。

{{< hint "note" >}}托管资源依靠轮询来检测外部系统的变化。{{< /hint >}}

Crossplane 会对所有资源进行双重检查，以确认它们是否处于所需的状态。 Crossplane 默认每隔一小时进行一次检查。 使用 `--sync-interval` Crossplane pod 参数可更改这一间隔。

最大调节速率 "定义了 crossplane 调节资源的速率（以每秒次数为单位）。

降低"--max-reconcile-rate"（最大同步率）或使其更小，可以减少 crossplane 被引用的 CPU 资源，但会增加更改后的资源完全同步所需的时间。

增加"-max-reconcile-rate"，或者使其变大，会增加 crossplane 被引用的 CPU 资源，但可以让 crossplane 更快地调节所有资源。

{{< hint "important" >}}大多数 Provider 使用自己的"-max-reconcile-rate"（最大重合率）。 这将决定 Provider 及其管理资源的相同设置。 将"-max-reconcile-rate "引用到 crossplane 只能控制核心 crossplane 资源的速率。{{< /hint >}}

##### 实现实时合成

启用实时 Composition 后，crossplane 会使用 Kubernetes 观察器来观察每个组成资源。 当组成资源发生变化时，crossplane 会从 Kubernetes API 服务器接收事件。 例如，当 Provider 将 "Ready "条件设置为 "true "时。

{{<hint "important" >}}实时 Composition 是一项 alpha 功能，默认情况下不会启用。{{< /hint >}}

启用实时 Composition 后，crossplane 不会被引用"--poll-interval "设置。

通过 [更改 crossplane pod 设置]( ) 启用实时 Composition 支持。{{<ref "./pods#change-pod-settings">}}) 并启用{{<hover label="deployment" line="12">}}-启用-enable-realtime-composition{{</hover>}}参数。

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
        - --enable-realtime-compositions
```

{{<hint "tip" >}}

crossplane 安装指南]({{<ref "../software/install#feature-flags">}})介绍了如何启用功能标志，如{{<hover label="deployment" line="12">}}--启用实时 Composition{{</hover>}}等功能标志。{{< /hint >}}

##### 对账重试率

最大校正率 "设置可配置 crossplane 或 Providers 每秒尝试校正资源的次数。 默认值为每秒 10 次。

所有核心 crossplane 组件共享调和速率。 每个 Provider 实现自己的最大调和速率设置。

##### 调解员人数

第二个值"-max-reconcile-rate "定义的是 crossplane 一次可以对账的资源数量。 如果资源数量超过配置的"-max-reconcile-rate"，剩余资源必须等待，直到 crossplane 对账一个现有资源。

请阅读[更改 Pod 设置]({{<ref "#change-pod-settings">}}) 部分，了解应用这些设置的说明。

<!-- vale Microsoft.HeadingAcronyms = NO -->

<!-- allow 'RBAC' since that's the name -->

## RBAC 管理器 pod

<!-- vale Microsoft.HeadingAcronyms = YES -->

Crossplane RBAC 管理器 pod 可自动为 Crossplane 和 Crossplane Providers 设置所需的 Kubernetes RBAC 权限。

{{<hint "note" >}}crossplane 默认安装并启用 RBAC 管理器。 禁用 RBAC 管理器需要手动定义 Kubernetes 权限，以便正确操作 crossplane。

RBAC 管理器设计文档](https://github.com/crossplane/crossplane/blob/master/design/design-doc-rbac-manager.md) 提供了有关 crossplane RBAC 要求的更全面的详细信息。{{< /hint >}}

### 禁用 RBAC 管理器

从 `crossplane-system` 名称空间中删除 `crossplane-rbac-manager` 部署，从而在安装后禁用 RBAC 管理器。

通过编辑 Helm `values.yaml` 文件，将 `rbacManager.deploy` 设置为 `false`，在安装前禁用 RBAC 管理器。

{{< hint "note" >}}

有关在安装过程中更改 Crossplane pod 设置的说明，请参阅 [Crossplane Install]({{<ref "../software/install">}}) 部分。{{< /hint >}}

<!-- vale Microsoft.HeadingAcronyms = NO -->

<!-- allow 'RBAC' since that's the name -->

### RBAC 启动容器

<!-- vale Microsoft.HeadingAcronyms = YES -->

RBAC 管理器启动前需要 "CompositeResourceDefinition "和 "ProviderRevision "资源可用。

在启动主 RBAC 管理器容器之前，RBAC 管理器 init 容器会等待这些资源。

### RBAC 管理器容器

RBAC 管理器容器执行以下任务: 

* 创建 RBAC 角色并将其与 Provider 服务账户绑定，使其能够控制受管资源
* 允许 "crossplane "服务帐户创建受管资源
* 创建群集角色，以访问所有 namespace 中的 crossplane 资源

使用 [ClusterRoles]({{<ref "#crossplane-clusterroles">}}) 授予对集群中所有 crossplane 资源的访问权限。

#### Crossplane ClusterRoles

RBAC 管理器会创建四个 Kubernetes 集群角色（Kubernetes ClusterRoles）。 这些角色授予对整个集群的 crossplane 资源的权限。

<!-- vale Google.Headings = NO -->

<!-- disable heading checking for the role names -->

<!-- vale Google.WordList = NO -->

<!-- allow "admin" -->

##### crossplane-admin

<!-- vale Google.WordList = YES -->

<!-- vale Crossplane.Spelling = NO -->

crossplane-admin "群集角色具有以下权限: 

* 完全访问所有 crossplane 类型
* 完全访问所有 secrets 和 namespace（甚至是与 crossplane 无关的 secrets 和 namespace）
* 对所有集群 RBAC 角色、自定义资源定义和事件的只读访问权限
* 将 RBAC 角色与其他实体绑定的能力。

<!-- vale Crossplane.Spelling = YES -->

查看完整的 RBAC 策略

```shell
kubectl describe clusterrole crossplane-admin
```

##### crossplane-edit

crossplane-edit` ClusterRole 拥有以下权限: 

* 完全访问所有 crossplane 类型
* 完全访问所有秘密（即使是与 Crossplane 无关的秘密）
* 只读访问所有 namespace 和事件（即使与 crossplane 无关）。

查看完整的 RBAC 策略

```shell
kubectl describe clusterrole crossplane-edit
```

##### crossplane-view

crossplane-view "ClusterRole 具有以下权限: 

* 对所有 crossplane 类型的只读访问权限
* 以只读方式访问所有 namespace 和事件（即使与 crossplane 无关）。

查看完整的 RBAC 策略

```shell
kubectl describe clusterrole crossplane-view
```

##### crossplane-browse

crossplane-browse "ClusterRole 具有以下权限: 

* 只读访问 crossplane 成分和 XRD。这样，资源索赔创建者就可以发现并选择合适的成分。

查看完整的 RBAC 策略

```shell
kubectl describe clusterrole crossplane-browse
```

## 领导选举

默认情况下，集群中只运行一个 Crossplane pod。 如果运行的 Crossplane pod 多于一个，则两个 pod 都会尝试管理 Crossplane 资源。 为防止冲突，Crossplane 采用 "选举领导者 "的方式，每次只由一个 pod 控制。 其他 Crossplane pod 处于备用状态，直到领导者失效。

{{< hint "note" >}}可以运行多个 crossplane 或 RBAC 管理器 pod 以实现冗余。

大多数部署都不需要冗余 pod。{{< /hint >}}

crossplane pod 和 RBAC 管理器 pod 都支持领导者选举。

使用 `--leader-election` pod 参数启用领导人选举。

{{< hint "warning" >}}

<!-- vale write-good.TooWordy = NO -->

<!-- "multiple" -->

<!-- vale write-good.Passive = NO -->

<!-- allow "is unsupported" -->

不支持在不进行领导者选举的情况下运行多个 crossplane pod。

<!-- vale write-good.Passive = YES -->

<!-- vale write-good.TooWordy = YES -->

{{< /hint >}}

## 更改 pod 设置

在安装 crossplane 之前，通过编辑 Helm `values.yml` 文件更改 pod 设置；或者在安装之后，通过编辑 `Deployment` 更改 pod 设置。

配置选项]({{<ref "../software/install#customize-the-crossplane-helm-chart">}})和[功能标志]({{<ref "../software/install#customize-the-crossplane-helm-chart">}}配置选项]( ) 和[功能标志]( ) 的完整列表可在[crossplane install]({{<ref "../software/install">}}) 部分。

{{< hint "note" >}}

有关在安装过程中更改 Crossplane pod 设置的说明，请参阅 [Crossplane Install]({{<ref "../software/install">}}) 部分。{{< /hint >}}

#### 编辑部署

{{< hint "note" >}}这些设置同时适用于 `crossplane` 和 `rbac-manager` pod 以及 `Deployments`。{{< /hint >}}

要更改已安装的 crossplane pod 的设置，请使用命令编辑 `crossplane-system` 名称空间中的 `crossplane` 部署

kubectl edit deployment crossplane --namespace crossplane-system

{{< hint "warning" >}}更新 crossplane 部署会重启 crossplane pod。{{< /hint >}}

将 crossplane pod 参数添加到{{<hover label="args" line="9" >}}spec.template.spec.containers[].args{{< /hover >}}部分。

例如，要更改 "同步间隔"，请添加{{<hover label="args" line="12" >}}--同步间隔=30m{{< /hover >}}.

```yaml {label="args", copy-lines="1"}
kubectl edit deployment crossplane --namespace crossplane-system
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
        - --sync-interval=30m
```

#### 使用环境变量

核心 crossplane pod 会在启动时检查配置的环境变量，以更改默认设置。

可配置环境变量的完整列表请参见 [Crossplane Install]({{<ref "../software/install">}}) 部分。
