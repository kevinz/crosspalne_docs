---

title: Claims
weight: 60
description: "Claims是一种通过名称空间范围来消费 Crossplane 资源的方式"

---

claims 将一组受管资源表示为一个命名空间内的单一 Kubernetes 对象。

用户在访问 CompositeResourceDefinition 中定义的自定义 API 时会创建 claims。

{{< hint "tip" >}}

claims 类似于 [composite resources]({{<ref "./composite-resources">}}Claims 和复合资源的区别在于，crossplane 可以在一个 namespace 中创建 Claims，而复合资源是集群作用域的。{{< /hint >}}

{{<expand "Confused about Compositions, XRDs, XRs and Claims?" >}}crossplane 有四个核心组件，用户通常会把它们混为一谈: 

* [Composition]({{<ref "./compositions">}}) - 用于定义如何创建资源的模板。
* [Composition Resource Definition]({{<ref "./composite-resource-definitions">}}) (`XRD`) - 一种自定义 API 规范。
* [复合资源]({{<ref "./composite-resources">}}) (`XR`) - 通过使用 Composition Resource Definition 中定义的自定义 API 创建。XRs 使用 Composition 模板来创建新的托管资源。
* claims (`XRC`) - 本页面。与 Composition Resource 类似，但具有名称空间范围。

{{</expand >}}

## 创建claim

创建claim需要一个 [Composition]({{<ref "./compositions">}}) 和 [CompositeResourceDefinition]({{<ref "./composite-resource-definitions">}}) (`XRD`)已经安装。

{{<hint "note" >}}XRD 必须 [启用claim]({{<ref "./composite-resource-definitions#enable-claims">}}).{{< /hint >}}

Composition 定义了要创建的资源集。 XRD 定义了用户为请求资源集而调用的自定义 API。

Crossplane组件关系图](/media/composition-how-it-works.svg)

例如，该 {{<hover label="xrd1" line="2">}}复合资源定义{{</hover>}}创建了一个 Composition 资源 API 端点{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}并启用claim API 端点{{<hover label="xrd1" line="11">}}数据库.example.org{{</hover>}}

```yaml {label="xrd1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: XMyDatabase
    plural: xmydatabases
  claimNames:
    kind: Database
    plural: databases
  # Removed for brevity
```

claim被引用 XRD 的{{<hover label="xrd1" line="11">}}种类{{</hover>}}API 端点来请求资源。

claim的 {{<hover label="xrd1" line="1">}}版本{{</hover>}}与 XRD {{<hover label="xrd1" line="6">}}组{{</hover>}}和{{<hover label="claim1" line="2">}}类型{{</hover>}}与 XRD{{<hover label="xrd1" line="11">}}claim名称种类{{</hover>}}

```yaml {label="claim1",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  # Removed for brevity
```

当用户在 namespace 中创建一个 Composition 时，crossplane 也会创建一个复合资源。

被引用 {{<hover label="claimcomp" line="1">}}kubectl describe{{</hover>}}来查看相关 Composition 资源。

资源参考 {{<hover label="claimcomp" line="6">}}资源参考{{</hover>}}是为该claim创建的 Composition 资源 crossplane。

```shell {label="claimcomp",copy-lines="1"}
kubectl describe database.example.org/my-claimed-database
Name:         my-claimed-database
API Version:  example.org/v1alpha1
Kind:         database
Spec:
  Resource Ref:
    API Version:  example.org/v1alpha1
    Kind:         XMyDatabase
    Name:         my-claimed-database-rr4ll
# Removed for brevity.
```

被引用 {{<hover label="getcomp" line="1">}}kubectl describe{{</hover>}}在 Composition 资源上查看{{<hover label="getcomp" line="6">}}claims Ref{{</hover>}}将复合资源与原始 claims 链接起来。

```shell {label="getcomp",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-rr4ll
Name:         my-claimed-database-rr4ll
API Version:  example.org/v1alpha1
Kind:         XMyDatabase
Spec:
  Claim Ref:
    API Version:  example.org/v1alpha1
    Kind:         database
    Name:         my-claimed-database
    Namespace:    default
```

{{<hint "note" >}}Crossplane 支持直接创建复合资源。 Composition 允许为使用自定义 API 的用户提供名称空间范围和隔离。

如果您在 Kubernetes 部署中不使用 namespace，则不需要claim。{{< /hint >}}

### claim现有 Composition 资源

默认情况下，创建一个 Composition 会创建一个新的复合资源。 Claims 也可以链接到现有的复合资源。

claim现有复合资源的一个用例可能是资源调配缓慢。 复合资源可以预先调配，claim可以使用这些资源，而无需等待创建。

设置claim的 {{<hover label="resourceref" line="6">}}资源{{</hover>}}并匹配预先存在的 Composition 资源{{<hover label="resourceref" line="9">}}名称{{</hover>}}.

```yaml {label="resourceref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  resourceRef:
    apiVersion: example.org/v1alpha1
    kind: XMyDatabase
    name: my-pre-created-xr
```

如果一个 claims 指定了一个{{<hover label="resourceref" line="6">}}资源{{</hover>}}则 crossplane 不会创建复合资源。

{{<hint "note" >}}所有 claims 都有一个{{<hover label="resourceref" line="6">}}资源编号。{{</hover>}}手动定义{{<hover label="resourceref" line="6">}}资源编号{{</hover>}}无需手动定义。{{<hover label="resourceref" line="6">}}资源{{</hover>}}的信息。{{< /hint >}}

## claim connection secrets

如果claim需要连接secret，claim必须定义一个{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}对象。

写入{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}对象定义了用于保存连接详细信息的 Kubernetes secret 对象的名称。

{{<hint "note" >}}crossplane 会在与 claim 相同的 namespace 中创建 secret 对象。{{< /hint >}}

例如，将一个名为{{<hover label="claimSec" line="7">}}my-claim-secret 的新secret对象。{{</hover>}}被引用{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}的{{<hover label="claimSec" line="7">}}name: my-claim-secret{{</hover>}}.

```yaml {label="claimSec"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  writeConnectionSecretToRef:
    name: my-claim-secret
```

有关连接secret的更多信息，请阅读[连接secret知识库文章]({{<ref "/knowledge-base/guides/connection-details">}}).
