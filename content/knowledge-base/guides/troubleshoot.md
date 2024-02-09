---

title: 故障排除
weight: 306

---

## 请求的资源未找到

如果您使用 crossplane CLI 安装 "Provider "或 "Configuration"（例如，"crossplane install provider xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0" ），并收到 "服务器找不到请求的资源 "的错误，这通常表明您使用的 Crossplane CLI 已经过时。 换句话说，某些 Crossplane API 已从 alpha 升级到 beta 或稳定版，而旧插件并没有意识到这一变化。

## 资源状况和条件

大多数 crossplane 资源都有一个 "status "部分，可以表示该特定资源的当前状态。 对一个 crossplane 资源运行 "kubectl describe "经常会提供有关其状态的有深度的信息。 例如，要确定一个 GCP `CloudSQLInstance` 受管资源的状态，请使用该资源的 "kubectl describe"。

```shell {copy-lines="1"}
kubectl describe cloudsqlinstance my-db
Status:
  Conditions:
    Last Transition Time:  2019-09-16T13:46:42Z
    Reason:                Creating
    Status:                False
    Type:                  Ready
```

大多数 crossplane 资源都设置了 "Ready "条件。"Ready "代表资源的可用性--是否正在创建、删除、可用、不可用、绑定等。

## 资源活动

当有趣的事情发生时，大多数 crossplane 资源都会发出_events_。 你可以通过运行 `kubectl describe` 查看与资源相关的事件，例如 `kubectl describe cloudsqlinstance my-db`。 你也可以通过运行 `kubectl get events` 查看特定 namespace 中的所有事件。

```console
Events:
  Type Reason Age From Message
  ----     ------                   ----               ----                                                   -------
  Warning CannotConnectToProvider 16s (x4 over 46s)  managed/postgresqlserver.database.azure.crossplane.io cannot get referenced ProviderConfig: ProviderConfig.azure.crossplane.io "default" not found
```

&gt; 请注意，事件是有命名空间的，而许多 crossplane 资源（XRs 等）&gt; 是集群作用域的。 crossplane 会将集群作用域资源的事件发送到 &gt;"默认 "命名空间。

## crossplane logging

要获取更多信息或调查故障，下一个地方就是 Crossplane pod 日志，它应在 `crossplane-system` 名称空间中运行。 要获取当前的 Crossplane 日志，请运行以下命令: 

```shell
kubectl -n crossplane-system logs -lapp=crossplane
```

&gt; 请注意，默认情况下，crossplane 很少发布 logging - 事件通常是查找有关 Crossplane 正在做什么的信息的最佳 &gt; 地方。 如果你找不到要找的信息，你可能需要 &gt; 用 `--debug` flag 重启 Crossplane。

## Provider Logging

请记住，Crossplane 的大部分功能都是由 Provider 提供的。 您也可以使用 `kubectl logs` 查看 Provider 的日志。 按照惯例，它们默认情况下也很少发布日志。

```shell
kubectl -n crossplane-system logs <name-of-provider-pod>
```

所有由 crossplane 社区维护的 Provider 都会映射 Crossplane 对 `--debug` flag 的支持。 在 Provider 上设置 flag 的最简单方法是创建一个 `ControllerConfig` 并从 `Provider` 中引用它: 

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: debug-config
spec:
  args:
    - --debug
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: debug-config
```

&gt; 请注意，可以将对 `ControllerConfig` 的引用添加到已安装的 `Provider` 中，它会相应地更新其 `Deployment` 。

## Composition 和复合资源定义

### 一般故障排除步骤

Crossplane 及其 Providers 会将大多数错误信息记录到资源的事件字段中。 如果您的 Composition 资源没有得到配置，请按照以下步骤操作: 

1.使用 `kubectl describe` 或 `kubectl get event` 获取根资源的事件
2.如果事件中有错误，请处理。
3.如果没有错误，则跟踪其子资源。
    `kubectl get<KIND> <NAME> -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq`.
4.对返回的每个资源重复此过程。

{{< hint "note" >}}本节其余部分将向您展示如何在不使用外部工具的情况下调试与 Composition 相关的问题。 如果您正在使用带有用户界面的 ArgoCD 或 FluxCD，您可以在用户界面中可视化对象关系。 您也可以使用 kube-lineage 插件在终端中可视化对象关系。{{< /hint >}}

### 示例

#### Composition

<!-- vale Google.WordList = NO -->

您使用索赔部署了一个示例应用程序。 Kind = `ExampleApp`. Name = `example-application`.

如下图所示，示例应用程序从未达到可用状态。

1.查看索赔。
    ```bash
    kubectl describe exampleapp example-application
    
    状态: 
    条件: 
        最后转换时间: 2022-03-01T22:57:38Z
        原因:                 复合资源索赔正在等待复合资源就绪
        状态:                 false
        类型:                   就绪
    事件                   <none>
    ```
2.如果索赔没有错误，请检查索赔的 `.spec.resourceRef` 字段。
    ```bash
    kubectl get exampleapp example-application -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq
    
    {
      "apiVersion": "awsblueprints.io/v1alpha1"、
      "kind": "XExampleApp"、
      "名称": "example-application-xqlsz"（示例应用-xqlsz
    }
    ```
3.在前面的输出中，您可以看到此索赔的集群作用域资源。Kind = `XExampleApp` name = `example-application-xqlsz`
4.查看集群作用域资源的事件。
    ```bash
    kubectl describe xexampleapp example-application-xqlsz
    
    事件: 
    类型 原因 年龄 来自 消息
    ---- ------ ---- ---- -------
    Normal PublishConnectionSecret 9s (x2 over 10s) defined/compositeresourcedefinition.apiextensions.crossplane.io 成功发布连接详情
    正常 SelectComposition 6 秒（x6，超过 11 秒） defined/positeresourcedefinition.apiextensions.crossplane.io 成功选择 Composition
    警告 ComposeResources 6 秒（x6 超过 10 秒）defined/compiteresourcedefinition.apiextensions.crossplane.io 无法从索引 3 处的资源模板渲染组成资源: 无法使用干运行创建为组成资源命名: 创建过程中可能未设置空名称空间
    正常 ComposeResources 6s (x6 over 10s) defined/compositeresourcedefinition.apiextensions.crossplane.io 成功组成资源
    ```
5.你在事件中看到了错误。它在抱怨没有在其 Composition 中指定 namespace。对于这种特殊错误，您可以获取其子资源并检查哪个没有创建。
    ```bash
    kubectl get xexampleapp example-application-xqlsz -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq
    
    [
        {
            "apiVersion": "awsblueprints.io/v1alpha1"、
            "类型": "XDynamoDBTable"XDynamoDBTable"、
            "名称": "example-application-xqlsz-6j9nm
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1"、
            "类型": "XIAMPolicy"XIAMPolicy"、
            "名称": "example-application-xqlsz-lp9wt"（示例应用-xqlsz-lp9wt
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1"、
            "类型": "XIAMPolicy"XIAMPolicy"、
            "名称": "example-application-xqlsz-btwkn"（示例应用-xqlsz-btwkn
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1"、
            "kind": "IRSA
        }
    ]
    ```
6.注意数组中的最后一个元素没有名称。当 Composition 中的资源验证失败时，资源对象没有创建，也没有名称。对于这个特殊问题，必须指定 IRSA 资源的 namespace。

#### Composition 资源定义

调试 Composite Resource Definition (XRD) 就像调试 Composition 一样。

1.获取 XRD
    ```bash
    kubectl get xrd testing.awsblueprints.io
    
    名称 建立时间
    testing.awsblueprints.io 66s
    ```
2.请注意它的状态不是已建立。您可以描述此 XRD 以获取其事件。
    ```bash
    kubectl describe xrd testing.awsblueprints.io
    
    事件: 
    类型 原因 年龄 来自 消息
    ---- ------ ---- ---- -------
    正常 ApplyClusterRoles 3m19s (x3 over 3m19s) rbac/compiteresourcedefinition.apiextensions.crossplane.io Applied RBAC ClusterRoles
    正常 RenderCRD 18 秒（x9，超过 3m19s） defined/compositeresourcedefinition.apiextensions.crossplane.io 渲染复合资源 CustomResourceDefinition
    Warning EstablishComposite 18s (x9 over 3m19s) defined/compiteresourcedefinition.apiextensions.crossplane.io 无法应用已渲染的复合资源 CustomResourceDefinition: can't create object: CustomResourceDefinition.apiextensions.k8s.io "testing.awsblueprints.io" 无效:  metadata.name: Invalid value: "testing.awsblueprints.io": 必须是 spec.names.plural+". "+spec.group
    ```
3.从事件中可以看到，crossplane 无法为该 XRD 生成相应的 CRD。在这种情况下，请确保名称是 `spec.names.plural+". "+spec.group`

#### Providers

您可以通过两种方式使用安装提供程序: "configuration.pkg.crossplane.io "和 "provider.pkg.crossplane.io"。 您可以使用其中一种方式安装提供程序，与提供程序本身在功能上没有区别。 如果您定义了一个 "configuration.pkg.crossplane.io "对象，Crossplane 就会创建一个 "provider.pkg.crossplane.io "对象并对其进行管理。 请参考[软件包文档]({{<ref "/master/concepts/packages">}}) 获取有关 crossplane 包的更多信息。

如果您遇到 Provider 问题，以下步骤是一个很好的起点。

1.检查 Provider 对象的状态。
    ```bash
    kubectl describe provider.pkg.crossplane.io provider-aws
    
    状态: 
        条件: 
            最后转换时间: 2022-08-04T16:19:44Z
            原因: HealthyPackageRevision
            状态:                 True
            类型:                   健康
            最后转换时间: 2022-08-04T16:14:29Z
            原因 活动包修订
            状态:                 True
            类型: 已安装 已安装
        当前标识符: crossplane/provider-aws:v0.29.0
        当前修订: provider-aws-a2e16ca2fc1a
    事件: 
        类型 原因 年龄 来自 信息
        ---- ------ ---- ---- -------
        Normal InstallPackageRevision 9m49s (x237 over 4d17h) packages/provider.pkg.crossplane.io Successfully installed package revision
    ````在上面的输出中，您可以看到该 Provider 是健康的。要获取有关此 Provider 的更多信息，可以深入挖掘。当前修订版本 "字段让你知道下一个要查看的对象。
2.创建 Provider 对象时，crossplane 会根据 OCI 镜像的内容创建一个 "ProviderRevision "对象。在本例中，你指定的 OCI 镜像是 `crossplane/provider-aws:v0.29.0`。该镜像包含一个 YAML 文件，其中定义了 Kubernetes 对象，如部署、ServiceAccount 和 CRD。

ProviderRevision" 对象会根据 YAML 文件的内容创建提供程序运行所需的资源。 要检查作为提供程序包一部分部署的内容，可以检查 ProviderRevision 对象。 上面的 "当前修订 "字段表示该提供程序被引用的 ProviderRevision 对象。

```
```bash
kubectl get providerrevision provider-aws-a2e16ca2fc1a

NAME HEALTHY REVISION IMAGE STATE DEP-FOUND DEP-INSTALLED AGE
provider-aws-a2e16ca2fc1a True 1 crossplane/provider-aws:v0.29.0 Active 19d
```

When you describe the object, you find all CRDs managed by this object. 

```bash
kubectl describe providerrevision provider-aws-a2e16ca2fc1a

Status:
    Controller Ref:
        Name:  provider-aws-a2e16ca2fc1a
    Object Refs:
        API Version:  apiextensions.k8s.io/v1
        Kind:         CustomResourceDefinition
        Name:         natgateways.ec2.aws.crossplane.io
        UID:          5c36d1bc-61b8-44f8-bca0-47e368af87a9
        ....
Events:
    Type Reason Age From Message
    ----    ------             ----                   ----                                         -------
    Normal SyncPackage 22m (x369 over 4d18h)  packages/providerrevision.pkg.crossplane.io Successfully configured package revision
    Normal BindClusterRole 15m (x348 over 4d18h)  rbac/providerrevision.pkg.crossplane.io Bound system ClusterRole to provider ServiceAccount
    Normal ApplyClusterRoles 15m (x364 over 4d18h)  rbac/providerrevision.pkg.crossplane.io Applied RBAC ClusterRoles
```

The event field also indicates any issues that may have occurred during this process.
<!-- vale Google.WordList = YES -->
```

3.如果在上面的事件字段中没有看到任何错误，则应检查 crossplane 是否调配了部署及其状态。
    ```bash
    kubectl get deployment -n crossplane-system
    
    name ready up-to-date available age
    crossplane 1/1 1 1 105d
    crossplane-rbac-manager 1/1 1 105d
    Provider-aws-a2e16ca2fc1a 1/1 1 19d
    
    kubectl get pods -n crossplane-system
    
    name ready status restarts age
    crossplane-54db688c8d-qng6b 2/2 运行中 0 4d19h
    crossplane-rbac-manager-5776c9fbf4-wn5rj 1/1 运行中 0 4d19h
    Provider-aws-a2e16ca2fc1a-776769ccbd-4dqml 1/1 运行中 0 4d23h
    ```如果有任何 pod 出现故障，请检查其日志并解决问题。

## 暂停 crossplane

有时，例如当你遇到一个错误时，如果你想阻止它主动尝试管理你的资源，暂停 crossplane 可能会很有用。 要暂停 crossplane 而不删除它的所有资源，运行以下命令即可简单地缩减它的部署规模: 

```bash
kubectl -n crossplane-system scale --replicas=0 deployment/crossplane
```

一旦您能够纠正问题或平息事态，您只需将其部署规模重新扩大，就可以解除 crossplane 的暂停状态: 

```bash
kubectl -n crossplane-system scale --replicas=1 deployment/crossplane
```

## 暂停 Providers

在排除故障或协调复杂的资源迁移时，也可以暂停 Provider。 创建并引用 "ControllerConfig "是缩减 Provider 的最简单方法，可以修改 "ControllerConfig "或移除引用以恢复其规模: 

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: scale-config
spec:
  replicas: 0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: scale-config
```

&gt; 请注意，可以将对 `ControllerConfig` 的引用添加到已安装的 `Provider` 中，它会相应地更新其 `Deployment` 。

## 当资源挂起时删除

crossplane 管理的资源会被自动清理，以免留下任何运行中的东西。 这是通过使用 Finalizer 来实现的，但在某些情况下，Finalizer 可以阻止 Kubernetes 对象被删除。

为了解决这个问题，我们基本上要给对象打上补丁，移除它的 Finalizer，这样就可以完全删除它了。 请注意，这并不一定会删除 crossplane 管理的外部资源，所以你需要去云计算 Providers 的控制台，在那里查找需要清理的残留资源。

一般情况下，可以使用此命令从对象中移除 Finalizer: 

```shell
kubectl patch <resource-type> <resource-name> -p '{"metadata":{"finalizers": []}}' --type=merge
```

例如，对于名为 "my-db "的 "CloudSQLInstance "托管资源（"database.gcp.crossplane.io"），可以使用以下命令移除其 Finalizer: 

```shell
kubectl patch cloudsqlinstance my-db -p '{"metadata":{"finalizers": []}}' --type=merge
```

## 技巧、窍门和故障排除

在本节中，我们将介绍使用 Composition 资源时的一些常用技巧、窍门和故障排除步骤。 如果您想找出 Composition 资源无法运行的原因，[故障排除][froubles-ref] 页面也提供了一些有用的信息。

### 解决索赔和 XR 的问题

crossplane 在很大程度上依赖状态条件和事件来排除故障。 您可以使用 `kubectl describe` 查看这两种状态条件和事件--例如: 

```console
# Describe the PostgreSQLInstance claim named my-db
kubectl describe postgresqlinstance.database.example.org my-db
```

根据 Kubernetes 的惯例，Crossplane 会在错误发生地附近保留错误。 这意味着，如果你的 claim 由于 "Composition "或组成资源的问题而没有准备就绪，你需要 "跟踪引用 "来找出原因。 你的 claim 只会告诉你 XR 尚未准备就绪。

参考文献: 

1.在你的 claims 上运行 `kubectl describe` 查找你的 XR，并查找它的 "Resource Ref"（又名 `spec.resourceRef`）。
2.在 XR 上运行 `kubectl describe`。您将在此发现所使用的 `Composition` 中存在的问题（如果有的话）。
3.如果没有问题，但你的 XR 似乎还没有准备就绪，请查看 "Resource Refs"（或 `spec.resourceRefs`），找到你的组成资源。
4.在每个引用的组成资源上运行 `kubectl describe` 以确定其是否准备就绪，以及遇到了哪些问题（如果有的话）。

<!-- Named Links -->