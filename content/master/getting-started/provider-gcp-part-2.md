---

title: GCP Quickstart Part 2 
weight: 120 
tocHidden: true 
aliases: 
  - /master/getting-started/provider-azure-part-3

---

{{< hint "important" >}}本指南是系列指南的第二部分。

[第一部分**]({{<ref "provider-gcp" >}}包括安装 crossplane 并将 Kubernetes 集群连接到 GCP。

{{< /hint >}}

本指南将指导您使用 crossplane 构建和访问自定义 API。

## 先决条件

* 完成 [quickstart part 1]({{<ref "provider-gcp" >}}) 将 Kubernetes 连接到 GCP。
* 一个 GCP 账户，该账户具有创建 GCP [存储桶](https://cloud.google.com/storage) 和 [发布/子主题](https://cloud.google.com/pubsub) 的权限。

{{<expand "Skip part 1 and just get started" >}}

1.添加 Crossplane Helm 软件源并安装 Crossplane。

```shell
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
&&
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

2.当crossplane吊舱完成安装并准备就绪时，应用 GCP

Provider.

```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0
EOF
```

3.用 GCP 服务账户创建名为 `gcp-credentials.json` 的文件

JSON 文件。

{{< hint "tip" >}}[GCP 文档](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) 提供了如何生成服务帐户 JSON 文件的信息。{{< /hint >}}

4.从 GCP JSON 文件创建 Kubernetes secret

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./gcp-credentials.json
```

5.创建 _ProviderConfig_

包括您的{{< hover label="providerconfig" line="7" >}}GCP 项目 ID{{< /hover >}}设置中。

{{< hint type="tip" >}}从 `gcp-credentials.json` 文件中的 `project_id` 字段查找 GCP 项目 ID。{{< /hint >}}

{{< editCode >}}

```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $@<PROJECT_ID>$@
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```

{{< /editCode >}}

{{</expand >}}

## 安装 PubSub Providers

第一部分只安装了 GCP 存储提供程序。 本部分将部署一个 PubSub Topic 和一个 GCP 存储桶。 首先安装 GCP PubSub Provider。

将新的 Provider 添加到集群中。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-pubsub
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-pubsub:v0.41.0
EOF
```

使用 `kubectl get Provider` 查看新的 PubSub Provider。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-gcp-pubsub True True xpkg.upbound.io/upbound/provider-gcp-pubsub:v0.41.0 39s
provider-gcp-storage True True xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0 13m
upbound-provider-family-gcp True True xpkg.upbound.io/upbound/provider-family-gcp:v0.41.0 12m
```

## 创建自定义应用程序接口

<!-- vale alex.Condescending = NO -->

Crossplane 允许您为用户构建自己的自定义 API，抽象掉有关云提供商及其资源的细节。 您可以随心所欲地让自己的 API 变得复杂或简单。

<!-- vale alex.Condescending = YES -->

下面是一个自定义 API 的示例。

```yaml {label="exAPI"}
apiVersion: database.example.com/v1alpha1
kind: NoSQL
metadata:
  name: my-nosql-database
spec: 
  location: "US"
```

与任何 Kubernetes 对象一样，API 也有一个{{<hover label="exAPI" line="1">}}版本{{</hover>}},{{<hover label="exAPI" line="2">}}类型{{</hover>}}和{{<hover label="exAPI" line="5">}}规格{{</hover>}}.

###定义组和版本

要创建自己的 API，首先要定义一个 [API 组](https://kubernetes.io/docs/reference/using-api/#api-groups) 和 [版本](https://kubernetes.io/docs/reference/using-api/#api-versioning)。

_group_ 可以是任何值，但通常的习惯是映射到完全合格的域名。

<!-- vale gitlab.SentenceLength = NO -->

版本显示 API 的成熟或稳定程度，并在更改、添加或删除 API 中的字段时递增。

<!-- vale gitlab.SentenceLength = YES -->

crossplane 并不要求特定的版本或特定的版本命名约定，但强烈建议遵循[Kubernetes API 版本指南](https://kubernetes.io/docs/reference/using-api/#api-versioning)。

* `v1alpha1` - 随时可能更改的新 API。
* `v1beta1` - 稳定的现有 API。不鼓励进行破坏性更改。
* `v1` - 没有破坏性更改的稳定 API。

本指南被引用的组是{{<hover label="version" line="1">}}database.example.com{{</hover>}}.

由于这是 API 的第一个版本，本指南被引用的版本为{{<hover label="version" line="1">}}v1alpha1{{</hover>}}.

```yaml {label="version",copy-lines="none"}
apiVersion: database.example.com/v1alpha1
```

### 定义一种

API 组是相关 API 的逻辑集合。 在一个组中，有代表不同资源的单个种类。

例如，"队列 "组可能有 "PubSub "和 "CloudTask "两种。

种类 "可以是任何东西，但必须是[UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects)。

该应用程序接口的类型是{{<hover label="kind" line="2">}}PubSub{{</hover>}}

```yaml {label="kind",copy-lines="none"}
apiVersion: queue.example.com/v1alpha1
kind: PubSub
```

### 定义规格

应用程序接口最重要的部分是模式。 模式定义了用户接受的输入。

该应用程序接口允许用户提供{{<hover label="spec" line="4">}}位置{{</hover>}}运行云资源的位置。

所有其他资源设置都不能由用户配置。 这使得 crossplane 可以执行任何策略和标准，而不用担心用户出错。

```yaml {label="spec",copy-lines="none"}
apiVersion: queue.example.com/v1alpha1
kind: PubSub
spec: 
  location: "US"
```

### 应用应用程序接口

crossplane 被引用{{<hover label="xrd" line="3">}}Composition 资源定义{{</hover>}}(也称为 "XRD"）在 Kubernetes 中安装您的自定义 API。

XRD {{<hover label="xrd" line="6">}}规范{{</hover>}}包含有关应用程序接口的所有信息，包括{{<hover label="xrd" line="7">}}组{{</hover>}},{{<hover label="xrd" line="12">}}版本{{</hover>}},{{<hover label="xrd" line="9">}}种类{{</hover>}}和{{<hover label="xrd" line="13">}}模式{{</hover>}}.

XRD 的 {{<hover label="xrd" line="5">}}名称{{</hover>}}必须是 {{<hover label="xrd" line="9">}}复数{{</hover>}}和{{<hover label="xrd" line="7">}}组{{</hover>}}.

模式 {{<hover label="xrd" line="13">}}模式{{</hover>}}被引用为{{<hover label="xrd" line="14">}}开放式应用程序接口 v3{{</hover>}}规范来定义应用程序接口 {{<hover label="xrd" line="17">}}规范{{</hover>}}.

应用程序接口定义了一个 {{<hover label="xrd" line="20">}}位置{{</hover>}}必须是 {{<hover label="xrd" line="22">}}之一{{</hover>}}或{{<hover label="xrd" line="23">}}欧盟{{</hover>}}或{{<hover label="xrd" line="24">}}美国{{</hover>}}.

应用此 XRD 在 Kubernetes 集群中创建自定义 API。

```yaml {label="xrd",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: pubsubs.queue.example.com
spec:
  group: queue.example.com
  names:
    kind: PubSub
    plural: pubsubs
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              location:
                type: string
                oneOf:
                  - pattern: '^EU$'
                  - pattern: '^US$'
            required:
              - location
    served: true
    referenceable: true
  claimNames:
    kind: PubSubClaim
    plural: pubsubclaims
EOF
```

添加 {{<hover label="xrd" line="29">}}claim名称{{</hover>}}允许用户在集群级别使用{{<hover label="xrd" line="9">}}pubsub{{</hover>}}端点访问该 API，或在名称空间中使用{{<hover label="xrd" line="29">}}pubsubclaim{{</hover>}}端点访问该 API。

namespace 范围内的 API 是 crossplane _Claim_。

{{<hint "tip" >}}有关 Composition 资源定义的字段和选项的更多详情，请阅读 [XRD 文档]({{<ref "../concepts/composite-resource-definitions">}}).{{< /hint >}}

使用 `kubectl get xrd` 查看已安装的 XRD。

```shell {copy-lines="1"}
kubectl get xrd
NAME ESTABLISHED OFFERED AGE
pubsubs.queue.example.com True True 7s
```

使用 `kubectl api-resources | grep pubsub` 查看新的自定义 API 端点

```shell {copy-lines="1",label="apiRes"}
kubectl api-resources | grep queue.example
pubsubclaims queue.example.com/v1alpha1 true PubSubClaim
pubsubs queue.example.com/v1alpha1 false PubSub
```

## 创建部署模板

当用户访问自定义 API 时，crossplane 会接收他们的输入，并将其与描述要部署的基础架构的模板相结合。 Crossplane 将此模板称为_Composition_。

composition {{<hover label="comp" line="3">}}Composition{{</hover>}}模板中的每个条目都是一个完整的资源定义，定义了所有资源设置和元数据，如标签和 Annotations。

该模板可创建一个 GCP{{<hover label="comp" line="10">}}存储{{</hover>}}{{<hover label="comp" line="11">}}存储桶{{</hover>}}和{{<hover label="comp" line="25">}}PubSub{{</hover>}}{{<hover label="comp" line="26">}}主题{{</hover>}}.

crossplane 被引用为 {{<hover label="comp" line="15">}}补丁{{</hover>}}将用户的输入应用到资源模板中。 该 Composition 将用户的{{<hover label="comp" line="16">}}位置{{</hover>}}输入，并将其作为{{<hover label="comp" line="14">}}位置{{</hover>}}被引用到单个资源中。

将此 Composition 应用于您的集群。

```yaml {label="comp",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: topic-with-bucket
spec:
  resources:
    - name: crossplane-quickstart-bucket
      base:
        apiVersion: storage.gcp.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            location: "US"
      patches:
        - fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "EU"
                US: "US"
    - name: crossplane-quickstart-topic
      base:
        apiVersion: pubsub.gcp.upbound.io/v1beta1
        kind: Topic
        spec:
          forProvider:
            messageStoragePolicy:
              - allowedPersistenceRegions: 
                - "us-central1"
      patches:
        - fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.messageStoragePolicy[0].allowedPersistenceRegions[0]"
          transforms:
            - type: map
              map: 
                EU: "europe-central2"
                US: "us-central1"
  compositeTypeRef:
    apiVersion: queue.example.com/v1alpha1
    kind: PubSub
EOF
```

复合类型 {{<hover label="comp" line="40">}}compositeTypeRef{{</hover >}}定义了哪些自定义 API 可以使用此模板创建资源。

{{<hint "tip" >}}请阅读 [Composition documentation]({{<ref "../concepts/compositions">}}) 获取更多关于配置 Composition 和所有可用选项的信息。

阅读[补丁和转换文档]({{<ref "../concepts/patch-and-transform">}})，了解有关 crossplane 如何使用补丁将用户输入映射到 Composition 资源模板的更多信息。{{< /hint >}}

使用 `kubectl get composition` 查看 Composition

```shell {copy-lines="1"}
kubectl get composition
NAME XR-KIND XR-APIVERSION AGE
topic-with-bucket PubSub queue.example.com 3s
```

## 访问自定义应用程序接口

安装了自定义应用程序接口（XRD）并与资源模板（Composition）关联后，用户就可以访问应用程序接口来创建资源。

创建一个 {{<hover label="xr" line="2">}}对象来创建云资源。{{</hover>}}对象来创建云资源。

```yaml {copy-lines="all",label="xr"}
cat <<EOF | kubectl apply -f -
apiVersion: queue.example.com/v1alpha1
kind: PubSub
metadata:
  name: my-pubsub-queue
spec: 
  location: "US"
EOF
```

使用 `kubectl get pubsub` 查看资源。

```shell {copy-lines="1"}
kubectl get pubsub
NAME SYNCED READY COMPOSITION AGE
my-pubsub-queue True True topic-with-bucket 2m12s
```

该对象是一个 crossplane _composite resource_（也称为 `XR`）。 它是一个单独的对象，代表从 Composition 模板创建的资源集合。

使用 `kubectl get managed` 查看单个资源

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED EXTERNAL-NAME AGE
topic.pubsub.gcp.upbound.io/my-pubsub-queue-cjswx True True my-pubsub-queue-cjswx 3m4s

NAME READY SYNCED EXTERNAL-NAME AGE
bucket.storage.gcp.upbound.io/my-pubsub-queue-vljg9 True True my-pubsub-queue-vljg9 3m4s
```

使用 `kubectl delete pubsub` 删除资源。

```shell {copy-lines="1"}
kubectl delete pubsub my-pubsub-queue
pubsub.queue.example.com "my-pubsub-queue" deleted
```

使用 `kubectl get managed` 验证 crossplane 是否删除了资源

{{<hint "note" >}}删除资源可能需要 5 分钟。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 使用带有 namespace 的应用程序接口

访问应用程序接口 `pubsub` 发生在集群范围内。 大多数组织将用户隔离到 namespace 中。

Crossplane _Claim_ 是名称空间中的自定义应用程序接口。

创建 _Claim_ 就像访问自定义 API 端点一样，只不过是使用了{{<hover label="claim" line="3">}}类型{{</hover>}}从自定义 API 的 "claim名称 "中创建。

创建一个新的命名空间，以测试在其中创建一个 claims。

```shell
kubectl create namespace crossplane-test
```

然后在 `crossplane-test` 名称空间中创建一个 claim。

```yaml {label="claim",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: queue.example.com/v1alpha1
kind: PubSubClaim
metadata:  
  name: my-pubsub-queue
  namespace: crossplane-test
spec: 
  location: "US"
EOF
```

使用 `kubectl get claim -n crossplane-test` 查看claim。

```shell {copy-lines="1"}
kubectl get claim -n crossplane-test
NAME SYNCED READY CONNECTION-SECRET AGE
my-pubsub-queue True True 2m10s
```

claims 会自动创建一个复合资源，该资源会创建托管资源。

使用 `kubectl get composite` 查看 Crossplane 创建的 Composition 资源。

```shell {copy-lines="1"}
kubectl get composite
NAME SYNCED READY COMPOSITION AGE
my-pubsub-queue-7bm9n True True topic-with-bucket 3m10s
```

同样，使用 `kubectl get managed` 查看托管资源。

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED EXTERNAL-NAME AGE
topic.pubsub.gcp.upbound.io/my-pubsub-queue-7bm9n-6kdq4 True True my-pubsub-queue-7bm9n-6kdq4 3m22s

NAME READY SYNCED EXTERNAL-NAME AGE
bucket.storage.gcp.upbound.io/my-pubsub-queue-7bm9n-hhwx8 True True my-pubsub-queue-7bm9n-hhwx8 3m22s
```

删除 claim 会删除所有 crossplane 生成的资源。

kubectl delete claim -n crossplane-test my-pubsub-queue

```shell {copy-lines="1"}
kubectl delete pubsubclaim my-pubsub-queue -n crossplane-test
pubsubclaim.queue.example.com "my-pubsub-queue" deleted
```

{{<hint "note" >}}删除资源可能需要 5 分钟。{{< /hint >}}

使用 `kubectl get composite` 验证 crossplane 是否删除了 Composition 资源。

```shell {copy-lines="1"}
kubectl get composite
No resources found
```

使用 `kubectl get managed` 验证 crossplane 是否删除了托管资源。

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 下一步

* 在[Provider CRD reference](https://marketplace.upbound.io/providers/upbound/provider-family-aws/)中探索 Crossplane 可以配置的 AWS 资源。
* 加入 [Crossplane Slack](https://slack.crossplane.io/) 并与 Crossplane 用户和贡献者联系。
* 阅读更多有关[Crossplane 概念]({{<ref "../concepts">}})，了解 Crossplane 还能做些什么。
