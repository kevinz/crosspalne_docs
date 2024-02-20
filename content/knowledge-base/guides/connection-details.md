---

title: 了解连接细节
weight: 11
description: "如何跨 crossplane 受管资源、复合资源、Composition 和 claims 创建和管理连接详情"

---

在 crossplane 中引用连接详情需要以下组件: 

* 在[claim]( ) 中定义`writeConnectionSecretToRef.name`。{{<ref "/master/concepts/claims#claim-connection-secrets">}}).
* 在 [Composition]() 中定义 `writeConnectionSecretsToNamespace` 值。{{<ref "/master/concepts/compositions#composite-resource-combined-secret">}}).
* 在 [Composition]( ) 中为每个资源定义 `writeConnectionSecretToRef` 名称和名称空间。{{<ref "/master/concepts/compositions#composed-resource-secrets">}}).
* 在 [Composition]( ) 中用 `connectionDetails` 定义每个composition资源产生的secret密钥列表。{{<ref "/master/concepts/compositions#define-secret-keys">}}).
* 可选择在 [CompositeResourceDefinition]( ) 中定义 `connectionSecretKeys` 。{{<ref "/master/concepts/composite-resource-definitions#manage-connection-secrets">}}).

{{<hint "note">}}本指南讨论创建 Kubernetes secret。 crossplane 还支持使用外部secret存储，如 [HashiCorp Vault](https://www.vaultproject.io/)。

请阅读[外部secret存储指南]({{<ref "/knowledge-base/integrations/vault-as-secret-store">}}) 以获取更多有关与外部secret存储一起使用 crossplane 的信息。{{</hint >}}

## 背景

当 [Provider]({{<ref "/master/concepts/providers">}}) 创建托管资源时，该资源可能会生成特定于资源的详细信息。 这些详细信息可能包括用户名、密码或 IP 地址等连接详细信息。

crossplane 将这些信息称为 _connection details_ 或 _connection secrets_。

Provider 定义了要将哪些信息作为受管资源的_连接详情_来展示。

<!-- vale gitlab.SentenceLength = NO -->

<!-- wordy because of type names -->

当一个受管资源是一个 [Composition]({{<ref "/master/concepts/compositions">}})、[composition资源定义]({{<ref "/master/concepts/composite-resource-definitions">}})和可选的[claim]({{<ref "/master/concepts/claims">}}) 定义了哪些细节是可见的以及存储在哪里。

<!-- vale gitlab.SentenceLength = YES -->

{{<hint "note">}}以下所有示例都被引用了同一组 Composition、CompositeResourceDefinitions 和 Claims。

所有示例都依赖 [upbound provider-aws-iam](https://marketplace.upbound.io/providers/upbound/provider-aws-iam/) 来创建资源。

{{<expand "Reference Composition" >}}

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xsecrettest.example.org
spec:
  writeConnectionSecretsToNamespace: other-namespace
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: XSecretTest
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            userSelector:
              matchControllerRef: true
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: attribute.secret
        - fromConnectionSecretKey: attribute.ses_smtp_password_v4
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret1"
    - name: user
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: User
        spec:
          forProvider: {}
    - name: user2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: User
        metadata:
          labels:
            docs.crossplane.io: user
        spec:
          forProvider: {}
    - name: key2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            userSelector:
              matchLabels:
                docs.crossplane.io: user
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
        - name: key2-password
          fromConnectionSecretKey: password
        - name: key2-secret
          fromConnectionSecretKey: attribute.secret
        - name: key2-smtp
          fromConnectionSecretKey: attribute.ses_smtp_password_v4
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret2"
```

{{</expand >}}

{{<expand "Reference CompositeResourceDefinition" >}}

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsecrettests.example.org
spec:
  group: example.org
  connectionSecretKeys:
    - username
    - password
    - attribute.secret
    - attribute.ses_smtp_password_v4
    - key2-user
    - key2-pass
    - key2-secret
    - key2-smtp
  names:
    kind: XSecretTest
    plural: xsecrettests
  claimNames:
    kind: SecretTest
    plural: secrettests
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
```

{{</ expand >}}

{{<expand "Reference Claim" >}}

```yaml
apiVersion: example.org/v1alpha1
kind: SecretTest
metadata:
  name: test-secrets
  namespace: default
spec:
  writeConnectionSecretToRef:
    name: my-access-key-secret
```

{{</expand >}}
{{</hint >}}

## 受管理资源中的连接secret

<!-- vale gitlab.Substitutions = NO -->

<!-- vale gitlab.SentenceLength = NO -->

<!-- under 25 words -->

当受管资源创建连接secret时，Crossplane 可以将secret写入 [Kubernetes secret]({{<ref "/master/concepts/managed-resources#publish-secrets-to-kubernetes">}})或[外部secret存储]({{<ref "/master/concepts/managed-resources#publish-secrets-to-an-external-secrets-store">}}).

<!-- vale gitlab.SentenceLength = YES -->

<!-- vale gitlab.Substitutions = YES -->

创建单个托管资源会显示资源创建的连接 secrets。

{{<hint "note" >}}请阅读 [managed resources]({{<ref "/master/concepts/managed-resources">}}) 文档，了解更多关于配置资源和存储单个资源连接secret的信息。{{< /hint >}}

例如，创建一个{{<hover label="mr" line="2">}}访问密钥{{</hover>}}资源，并将连接secret保存在名为{{<hover label="mr" line="12">}}my-accesskey-secret 的 Kubernetes secret 中。{{</hover>}}的{{<hover label="mr" line="11">}}默认{{</hover>}}namespace 中名为 my-accesskey-secret 的连接secret中。

```yaml {label="mr"}
apiVersion: iam.aws.upbound.io/v1beta1
kind: AccessKey
metadata:
    name: test-accesskey
spec:
    forProvider:
        userSelector:
            matchLabels:
                docs.crossplane.io: user
    writeConnectionSecretToRef:
        namespace: default
        name: my-accesskey-secret
```

查看 Kubernetes secret，查看托管资源的连接详情。 这包括一个{{<hover label="mrSecret" line="11">}}attribute.secret{{</hover>}},{{<hover label="mrSecret" line="12">}}attribute.ses_smtp_password_v4{{</hover>}},{{<hover label="mrSecret" line="13">}}密码{{</hover>}}和{{<hover label="mrSecret" line="14">}}用户名{{</hover>}}

```yaml {label="mrSecret",copy-lines="1"}
kubectl describe secret my-accesskey-secret
Name:         my-accesskey-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
password:                        40 bytes
username:                        20 bytes
```

Composition 和 CompositeResourceDefinitions 要求提供由资源生成的secret的确切名称。

### Composition 中的连接secret

Composition 中创建连接详情的资源仍会创建一个包含其连接详情的secret对象。 Crossplane 还会为每个复合资源生成另一个secret对象，其中包含所有已定义资源的secret。

例如，一个 Composition 定义了两个{{<hover label="comp1" line="9">}}访问键{{</hover>}}对象。 {{<hover label="comp1" line="9">}}访问键{{</hover>}}都会将连接 secret 写入 {{<hover label="comp1" line="15">}}名称{{</hover>}}内的 {{<hover label="comp1" line="14">}}namespace{{</hover>}}资源定义的名称空间内的{{<hover label="comp1" line="13">}}writeConnectionSecretToRef{{</hover>}}.

crossplane 还会为整个 Composition 创建一个 secret 对象，保存在由以下命令定义的 namespace 中{{<hover label="comp1" line="4">}}writeConnectionSecretsToNamespace 所定义的命名空间中。{{</hover>}}定义的命名空间中保存的整个组合体的secret对象，并使用一个由 crossplane 生成的名称。

```yaml {label="comp1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key1
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1-secret
    - name: key2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2-secret
    # Removed for brevity
```

应用claim后，查看 Kubernetes secrets，可以看到创建的三个 secret 对象。

secret{{<hover label="compGetSec" line="3">}}key1-secret{{</hover>}}来自资源{{<hover label="comp1" line="6">}}资源{{</hover>}},{{<hover label="compGetSec" line="4">}}key2-secret{{</hover>}}来自资源{{<hover label="comp1" line="16">}}key2{{</hover>}}.

crossplane 在 namespace 中创建了另一个secret{{<hover label="compGetSec" line="5">}}其他名称空间{{</hover>}}中的secret。

```shell {label="compGetSec",copy-lines="1"}
kubectl get secrets -A
NAMESPACE NAME TYPE DATA AGE
docs key1-secret connection.crossplane.io/v1alpha1 4 4s
docs key2-secret connection.crossplane.io/v1alpha1 4 4s
other-namespace 70975471-c44f-4f6d-bde6-6bbdc9de1eb8 connection.crossplane.io/v1alpha1 0 6s
```

虽然 Crossplane 创建了一个secret对象，但默认情况下，Crossplane 不会向该对象添加任何数据。

```yaml {copy-lines="none"}
kubectl describe secret 70975471-c44f-4f6d-bde6-6bbdc9de1eb8 -n other-namespace
Name:         70975471-c44f-4f6d-bde6-6bbdc9de1eb8
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
```

Composition 必须列出要为每个资源存储的连接secret。 使用被引用的{{<hover label="comp2" line="16">}}连接详情{{</hover>}}对象，定义资源创建的秘钥。

{{<hint "warning">}}您不能更改{{<hover label="comp2" line="16">}}连接详情{{</hover>}}您必须删除并重新创建 Composition 才能更改{{<hover label="comp2" line="16">}}连接细节{{</hover>}}.{{</hint >}}

```yaml {label="comp2",copy-lines="16-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: attribute.secret
        - fromConnectionSecretKey: attribute.ses_smtp_password_v4
    # Removed for brevity
```

应用 claims 后，复合资源 secret 对象中包含的密钥列表在{{<hover label="comp2" line="16">}}连接详情{{</hover>}}.

```shell {copy-lines="1"}
kubectl describe secret -n other-namespace
Name:         b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
username:                        20 bytes
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
password:                        40 bytes
```

{{<hint "important">}}如果某个键未列在{{<hover label="comp2" line="16">}}连接详情{{</hover>}}就不会存储在 secret 对象中。{{< /hint >}}

### 管理相互冲突的秘钥

如果资源产生了冲突的键，则应创建一个唯一的名称，并提供连接详情{{<hover label="comp3" line="25">}}名称{{</hover>}}.

```yaml {label="comp3",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
    - name: key2
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
```

secret 对象包含两个密钥、{{<hover label="comp3Sec" line="9">}}用户名{{</hover>}}和{{<hover label="comp3Sec" line="10">}}用户{{</hover>}}

```shell {label="comp3Sec",copy-lines="1"}
kubectl describe secret -n other-namespace
Name:         b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
username:                        20 bytes
key2-user:                       20 bytes
# Removed for brevity.
```

### Composite Resource Definitions 中的连接secret

CompositeResourceDefinition (`XRD`)可以限制哪些密钥会被放入组合密钥并提供给 Claim。

默认情况下，XRD 会将composition资源 `connectionDetails` 中列出的所有secret密钥写入组合secret对象。

限制传递给组合secret对象的密钥，并声明带有{{<hover label="xrd" line="4">}}连接秘钥{{</hover>}}对象的密钥。

在 {{<hover label="xrd" line="4">}}连接秘钥{{</hover>}}中列出了要创建的secret密钥名称。 crossplane 只会将列出的密钥添加到组合secret中。

{{<hint "warning">}}您不能更改{{<hover label="xrd" line="4">}}连接密钥{{</hover>}}您必须删除并重新创建 XRD 才能更改{{<hover label="xrd" line="4">}}连接秘钥{{</hover>}}.{{</hint >}}

例如，XRD 可以将secret限制为只有{{<hover label="xrd" line="5">}}用户名{{</hover>}},{{<hover label="xrd" line="6">}}密码{{</hover>}}和名为{{<hover label="xrd" line="7">}}key2-user{{</hover>}}键。

```yaml {label="xrd",copy-lines="4-12"}
kind: CompositeResourceDefinition
spec:
  # Removed for brevity.
  connectionSecretKeys:
    - username
    - password
    - key2-user
```

单个资源的secret包含 Composition 的 `connectionDetails` 中详细说明的所有资源。

```shell {label="xrdSec",copy-lines="1"}
kubectl describe secret key1 -n docs
Name:         key1
Namespace:    docs

Data
====
password:                        40 bytes
username:                        20 bytes
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
```

claims 的 secret 只包含 XRD 允许的密钥{{<hover label="xrd" line="4">}}连接秘钥{{</hover>}}字段允许的密钥。

```shell {label="xrdSec2",copy-lines="2"}
kubectl describe secret my-access-key-secret
Name:         my-access-key-secret

Data
====
key2-user:  20 bytes
password:   40 bytes
username:   20 bytes
```

## Secret objects

Composition 为每个资源创建一个secret对象，并创建一个额外的secret，其中包含所有资源的所有secret。

crossplane 会将资源secret对象保存在资源的{{<hover label="comp4" line="11">}}writeConnectionSecretToRef 所定义的位置。{{</hover>}}.

Crossplane 会将组合secret以 Crossplane 生成的名称保存在 Composition 的{{<hover label="comp4" line="4">}}writeConnectionSecretsToNamespace{{</hover>}}.

```yaml {label="comp4",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
    - name: key2
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
```

如果claim使用了secret，则该secret会存储在与claim相同的 namespace 中，其名称在claim的{{<hover label="claim3" line="7">}}writeConnectionSecretToRef 中定义的名称。{{</hover>}}.

```yaml {label="claim3",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: SecretTest
metadata:
  name: test-secrets
  namespace: default
spec:
  writeConnectionSecretToRef:
    name: my-access-key-secret
```

在应用claim后，Crossplane 会创建以下secret: 

* 声明的secret {{<hover label="allSec" line="3">}}My-access-key-secret{{</hover>}}  在 claims 的 {{<hover label="claim3" line="5">}}namespace{{</hover>}}.
* 第一个资源的 secret 对象、 {{<hover label="allSec" line="4">}}key1{{</hover>}}.
* 第二个资源的secret对象、 {{<hover label="allSec" line="5">}}密钥 2{{</hover>}}.
* 的 Composition 资源secret对象。  {{<hover label="allSec" line="6">}}other-namespace{{</hover>}}由 Composition 的 `writeConnectionSecretsToNamespace` 定义。

```shell {label="allSec",copy-lines="none"}
kubectl get secret -A
NAMESPACE NAME TYPE DATA AGE
default my-access-key-secret connection.crossplane.io/v1alpha1 8 29m
docs key1 connection.crossplane.io/v1alpha1 4 31m
docs key2 connection.crossplane.io/v1alpha1 4 31m
other-namespace b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a connection.crossplane.io/v1alpha1 8 31m
```
