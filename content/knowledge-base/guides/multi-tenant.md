---

title: 多租户 crossplane
weight: 240

---

本指南介绍了如何利用 Kubernetes 原语和云原生生态系统中兼容的策略执行项目，在多租户环境中有效使用 crossplane。

## TL;DR

多租户 Crossplane 环境中的基础架构运营商通常会利用组成和 Kubernetes RBAC 来定义轻量级、标准化的策略，以规定开发人员在请求基础架构时可获得的自助服务级别。 这主要是通过在命名空间范围内公开抽象资源类型、为命名空间内的团队和个人定义 "角色 "以及修补底层托管资源的 "spec.providerConfigRef "来实现的，以便在从每个命名空间调配资源时使用特定的 "ProviderConfig "和凭据。 规模较大的组织或拥有更复杂环境的组织可能会选择采用第三方策略引擎，或扩展到多个 Crossplane 集群。 以下各节将更详细地介绍上述每种情况。

* [简要说明](#tldr)
* [背景](#background)
    - [集群作用域受管资源](#cluster-scoped-managed-resources)
    - [名称空间作用域索赔](#namespace-scoped-claims)
* [单集群多租户](#single-cluster-multi-tenancy)
    - [作为隔离机制的 Composition](#composition-as-an-isolation-mechanism)
    - [作为隔离机制的 namespace](#namespaces-as-an-isolation-mechanism)
    - [使用 Open Policy Agent 执行策略](#policy-enforcement-with-open-policy-agent)
* [多集群多租户](#multi-cluster-multi-tenancy)
    - [带配置包的可重现平台](#reproducible-platforms-with-configuration-packages)
    - [控制平面的控制平面](#control-plane-of-control-planes)

## 背景

Crossplane 设计用于在多租户环境中运行，在这种环境中，许多团队都在使用集群中基础设施运营商提供的服务和抽象。 Crossplane 生态系统中的两大设计模式促进了这一功能的实现。

###集群范围内的受管资源

通常情况下，提供反映外部 API 的细粒度[托管资源]的 crossplane 提供商通过使用指向凭证源（如 Kubernetes 的 "Secret"、"Pod "文件系统或环境变量）的 "ProviderConfig "对象来进行身份验证。 然后，每个托管资源都会被引用一个 "ProviderConfig"，该对象指向具有足够权限来管理该资源类型的凭证。

例如，下面的 `provider-aws` 的 `ProviderConfig` 指向了具有 AWS 凭据的 Kubernetes `Secret`。

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: cool-aws-creds
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

如果用户希望将这些凭据用于配置 "RDSInstance"，则会在对象配置清单中引用 "ProviderConfig": 

```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: rdsmysql
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t3.medium
    masterUsername: masteruser
    allocatedStorage: 20
    engine: mysql
    engineVersion: "5.6.35"
    skipFinalSnapshotBeforeDeletion: true
  providerConfigRef:
    name: cool-aws-creds # name of ProviderConfig above
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: aws-rdsmysql-conn
```

由于 "ProviderConfig "和所有受管资源都是集群作用域，因此 "provider-aws "中的 RDS 控制器将通过获取 "ProviderConfig"、获取其指向的凭据并使用这些凭据来调和 "RDSInstance"，从而解析此引用。 这意味着，任何被赋予 [RBAC] 来管理 "RDSInstance "对象的人都可以使用任何凭据来这样做。 实际上，crossplane 假定只有作为基础架构管理员或平台构建者的人才会直接与集群作用域的资源交互。

### 名称空间范围索赔

受管资源存在于集群范围，而使用**CompositeResourceDefinition（XRD）**定义的复合资源则可能存在于集群或名称空间范围。 平台构建者定义的 XRD 和**Composition**可指定创建 XRD 实例时应创建哪些粒度的受管资源。 有关此架构的更多信息，请参阅 [Composition] 文档。

每个 XRD 都暴露在集群作用域中，但只有那些定义了 `spec.claimNames` 的 XRD 才会有一个 namespace 作用域变量。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xmysqlinstances.example.org
spec:
  group: example.org
  names:
    kind: XMySQLInstance
    plural: xmysqlinstances
  claimNames:
    kind: MySQLInstance
    plural: mysqlinstances
...
```

创建上述示例时，crossplane 会生成两个 [CustomResourceDefinitions]: 

1.集群作用域类型，具有 `kind: XMySQLInstance`。这被称为 ** Composition Resource (XR)**。
2.名称空间作用域类型，"kind: MySQLInstance`。这被称为 ** 索赔 (XRC)**。

平台构建者可以选择定义任意数量的 Composition 来映射这些类型，这意味着在给定名称空间中创建一个 `MySQLInstance`，就可以在集群范围内创建任意一组受管资源。 例如，创建一个 `MySQLInstance`，就可以创建上面定义的 `RDSInstance`。

## 单集群多租户

根据组织的规模和范围，平台团队可能会选择运行一个中央 crossplane 控制平面，也可能为每个团队或业务部门运行多个不同的控制平面。 本节将重点介绍在单个集群内为多个团队提供服务的情况，该集群可能是，也可能不是组织中许多其他 crossplane 集群之一。

###作为隔离机制的构成

托管资源总是反映底层 Providers API 公开的每一个字段，而 XRD 则可以拥有平台构建者选择的任何模式。 然后，XRD 模式中的字段可以修补到 Composition 中定义的底层托管资源中的字段上，本质上就是将这些字段作为可配置字段公开给 XR 或 XRC 的消费者。

This feature serves as a lightweight policy mechanism by only giving the
consumer the ability to customize the underlying resources to the extent the
platform builder desires. For instance, in the examples above, a platform
builder may choose to define a `spec.location` field in the schema of the
`XMySQLInstance` that is an enum with options `east` and `west`. In the
Composition, those fields could map to the `RDSInstance` `spec.region` field,
making the value either `us-east-1` or `us-west-1`. If no other patches were
defined for the `RDSInstance`, giving a user the ability (using RBAC) to create
a `XMySQLInstance` / `MySQLInstance` would be akin to giving the ability to
create a very specifically configured `RDSInstance`, where they can only decide
the region where it lives and they are restricted to two options.

这种模式与许多基础架构即代码工具形成鲜明对比，在后者中，终端用户必须拥有 Provider 凭据才能创建抽象呈现的底层资源。 Crossplane 采用不同的方法，在集群中定义各种凭据（使用 "ProviderConfig"），然后仅赋予 Provider 控制器使用这些凭据并代表用户配置基础架构的能力。 通过对 Kubernetes RBAC 进行标准化，即使使用具有不同 IAM 模型的许多 Provider，也能创建一致的权限模型。

### 作为隔离机制的 namespace

使用 Composition 定义抽象模式并将其修补为具体资源类型的功能固然强大，但在命名空间范围内定义 Claim 类型的功能则通过使 RBAC 能够应用命名空间限制而进一步增强了功能。 集群中的大多数用户都无法访问集群范围内的资源，因为 Kubernetes 和 crossplane 都认为这些资源只与基础设施管理员相关。

在我们简单的 `XMySQLInstance` / `MySQLInstance` 示例的基础上，平台构建者可以选择使用 `Role` 在命名空间范围内定义 `MySQLInstance` 的权限。 这样，用户就可以在其给定的命名空间中创建和管理 `MySQLInstance` ，但不能查看在其他命名空间中定义的 `MySQLInstance` 。

此外，由于 "metadata.namespace "是 XRC 上的一个字段，因此可以利用修补功能根据定义相应 XRC 的名称空间来配置受管资源。 如果平台构建者希望指定特定的凭据或一组凭据，让特定名称空间中的用户在使用 XRC 配置基础设施时可以使用，那么这一点就特别有用。 现在可以通过创建一个或多个在 "ProviderConfig "名称中包含名称空间名称的 "ProviderConfig "对象来实现这一点。 例如，如果在 "team-1 "名称空间中创建的任何 "MySQLInstance "应该在提供商控制器创建相应的 "RDSInstance "时使用特定的 AWS 凭据，平台构建者就可以: 

1.定义名称为 `team-1` 的 `ProviderConfig`。

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: team-1
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: team-1-creds
      key: creds
```

2.定义一个 "Composition"，将 XR 中的 Claim 引用的名称空间修补为 "RDSInstance "的 "providerConfigRef"。

```yaml
...
resources:
- base:
    apiVersion: database.aws.crossplane.io/v1beta1
    kind: RDSInstance
    spec:
      forProvider:
      ...
  patches:
  - fromFieldPath: spec.claimRef.namespace
    toFieldPath: spec.providerConfigRef.name
    policy:
      fromFieldPath: Required
```

这将导致 `RDSInstance` 被引用相应的 `MySQLInstance` 在哪个 namespace 中创建的 `ProviderConfig` 。

&gt; 请注意，该模型目前只允许在每个 &gt; 名称空间中使用一个 `ProviderConfig `。 不过，今后发布的 crossplane 版本应允许定义一组 &gt; `ProviderConfig `，可使用 [Multiple Source Field &gt; patching] 从中选择。

### 使用 Open Policy Agent 执行政策

在某些 Crossplane 部署模型中，仅使用组成和 RBAC 来定义策略不够灵活。 不过，由于 Crossplane 将外部基础设施的管理引入了 Kubernetes API，因此非常适合与云原生生态系统中的其他项目集成。 需要更强大策略引擎的组织和个人，或者只是喜欢使用更通用的语言来定义策略的组织和个人，通常会求助于[Open Policy Agent]（OPA）。 OPA 允许平台构建者使用[Rego]（一种特定领域的语言）编写自定义逻辑。 以这种方式编写策略，不仅可以纳入被评估的特定资源中的可用信息，还可以使用集群中代表的其他状态。 Crossplane 用户通常会安装 OPA 的[Gatekeeper]，以尽可能简化策略管理。

&gt; 可在 [此处] 观看 OPA 与 crossplane 一起使用的现场演示。

### 多集群多租户

在多个集群中部署 crossplane 的机构通常会利用两大功能，使多个控制平面的管理变得更加简单。

###可复制平台与配置包

[配置包]允许平台构建者将他们的 XRD 和 Composition 打包成[OCI 镜像]，这些镜像可以通过任何符合 OCI 标准的镜像注册中心进行分发。 这些包还可以声明对 Provider 的依赖关系，这意味着一个包就可以声明所有细粒度的托管资源、必须部署以调和这些资源的控制器，以及使用 Composition 公开相应的底层资源的抽象类型。

部署了许多 Crossplane 的企业利用配置包在每个集群中复制其平台。 这可以很简单，只需在安装 Crossplane 时加上自动安装配置包的 flag 即可。

```
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane --set configuration.packages='{"registry.upbound.io/xp/getting-started-with-aws:latest"}'
```

#### 控制平面的控制平面

在多集群多租户模式的基础上更进一步，一些企业选择使用一个中央 Crossplane 控制平面来管理它们的多个 Crossplane 集群。 这就需要设置中央集群，然后使用 Providers 来启动新的集群（例如使用 [provider-aws] 的 [EKS 集群]），然后使用 [provider-helm] 将 Crossplane 安装到新的远程集群中，并可能使用上述方法将通用配置包捆绑到每个安装中。

这种高级模式允许使用 Crossplane 本身对 Crossplane 集群进行全面管理，如果操作得当，它是一种可扩展的解决方案，可为单个组织内的许多租户提供专用控制平面。

<!-- Named Links -->

[受管资源]:  {{<ref "../../master/concepts/managed-resources" >}}[RBAC]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/ [Composition]:  {{<ref "/master/concepts/compositions" >}}[CustomResourceDefinitions]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/ [Open Policy Agent]: https://www.openpolicyagent.org/ [Rego]: https://www.openpolicyagent.org/docs/latest/policy-language/ [Gatekeeper]: https://open-policy-agent.github.io/gatekeeper/website/docs/ [here]: https://youtu.be/TaF0_syejXc [Multiple Source Field patching]: https://github.com/crossplane/crossplane/pull/2093 [Configuration packages]:  {{<ref "../../master/concepts/packages" >}}[OCI 镜像]: https://github.com/opencontainers/image-spec [EKS 集群]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws/latest/resources/eks.aws.crossplane.io/Cluster/v1beta1 [provider-aws]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws [provider-helm]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-helm/ [Open Service Broker API]: https://github.com/openservicebrokerapi/servicebroker [Crossplane Service Broker]: https://github.com/vshn/crossplane-service-broker [Cloudfoundry]: https://www.cloudfoundry.org/ [Kubernetes 服务目录]: https://github.com/kubernetes-sigs/service-catalog [vshn/application-catalog-demo]: https://github.com/vshn/application-catalog-demo
