---

title: 导入现有资源
weight: 200

---

如果您有已经在 Provider 中配置好的资源，您可以将它们作为受管资源导入，并让 Crossplane 对其进行管理。 受管资源的 [`managementPolicies`]({{<ref "/master/concepts/managed-resources#managementpolicies">}}) 字段可以将外部资源导入 crossplane。

crossplane 可以[手动]({{<ref "#import-resources-manually">}}) 或 [自动]({{<ref "#import-resources-automatically">}}).

## 手动导入资源

通过匹配托管资源中的 "crossplane.io/external-name "注解，crossplane 可以发现并导入现有的 Provider 资源。

要导入 Provider 中的现有外部资源，请创建一个带有 "crossplane.io/external-name "注解的新托管资源。 将注解值设置为 Provider 中的资源名称。

例如，要导入名为{{<hover label="annotation" line="5">}}的现有 GCP 网络{{</hover>}}的现有 GCP 网络，创建一个新的托管资源，并使用{{<hover label="annotation" line="5">}}my-existing-network{{</hover>}}在 Annotations 中使用。

```yaml {label="annotation",copy-lines="none"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  annotations:
    crossplane.io/external-name: my-existing-network
```

元数据 {{<hover label="name" line="5">}}元数据名称{{</hover>}}字段可以是任何你想要的内容，例如{{<hover label="name" line="5">}}导入的网络{{</hover>}}.

{{< hint "note" >}}该名称是 Kubernetes 对象的名称，与 Provider 内的资源名称无关。{{< /hint >}}

```yaml {label="name",copy-lines="none"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: imported-network
  annotations:
    crossplane.io/external-name: my-existing-network
```

将{{<hover label="fp" line="8">}}spec.forProvider{{</hover>}}字段为空。 crossplane 会导入设置并自动应用到托管资源。

{{< hint "important" >}}如果托管资源的{{<hover label="fp" line="8">}}spec.forProvider{{</hover>}}字段，则必须将其添加到 `forProvider` 字段中。

这些字段的值必须与 Provider 内的值一致，否则 crossplane 会覆盖现有值。{{< /hint >}}

```yaml {label="fp",copy-lines="all"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: imported-network
  annotations:
    crossplane.io/external-name: my-existing-network
spec:
  forProvider: {}
```

Crossplane 现在控制并管理该导入资源。 对管理资源 `spec` 的任何更改都会更改外部资源。

## 自动导入资源

通过 "观察"[管理策略]自动导入外部资源({{<ref "/master/concepts/managed-resources#managementpolicies">}}).

crossplane 导入只观察资源，但从不更改或删除资源。

{{<hint "important" >}}managed resource `managementPolicies`选项是测试版功能。

请参阅 "提供方 "的文件，查看 "提供方 "是否支持管理政策。{{< /hint >}}

<!-- vale off -->

###应用观察管理策略

<!-- vale on -->

创建一个与{{<hover label="oo-policy" line="1">}}apiVersion{{</hover>}}和{{<hover label="oo-policy" line="2">}}类型{{</hover>}}的受管资源，并添加{{<hover label="oo-policy" line="4">}}管理策略: ["观察"]{{</hover>}}添加到{{<hover label="oo-policy" line="3">}}规格{{</hover>}}

例如，要导入一个 GCP SQL 数据库实例，请创建一个新资源，并使用 {{<hover label="oo-policy" line="4">}}管理策略: ["观察"]。{{</hover>}}设置。

```yaml {label="oo-policy",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
spec:
  managementPolicies: ["Observe"]
```

### 添加外部名称 Annotations

添加 {{<hover label="oo-ex-name" line="5">}}crossplane.io/external-name{{</hover>}}该名称必须与 Provider 内部的名称一致。

例如，对于名为{{<hover label="oo-ex-name" line="5">}}我的外部数据库{{</hover>}}的 GCP 数据库，应用{{<hover label="oo-ex-name" line="5">}}crossplane.io/external-name{{</hover>}}Annotations 的值为{{<hover label="oo-ex-name" line="5">}}我的外部数据库{{</hover>}}.

```yaml {label="oo-ex-name",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
```

### 创建 Kubernetes 对象名称

创建一个 {{<hover label="oo-name" line="4">}}名称{{</hover>}}以用于 Kubernetes 对象。

例如，将 Kubernetes 对象命名为{{<hover label="oo-name" line="4">}}我的导入数据库{{</hover>}}.

```yaml {label="oo-name",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
```

### 确定特定的外部资源

如果 Provider 中有多个资源使用相同的名称，则应使用唯一的{{<hover line="9" label="oo-region">}}spec.forProvider{{</hover>}}字段来标识特定资源。

例如，只在{{<hover line="10" label="oo-region">}}us-central1{{</hover>}}区域的 GCP SQL 数据库。

```yaml {label="oo-region"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
  forProvider:
    region: "us-central1"
```

### 应用管理的资源

应用新的托管资源。 crossplane 会将云中外部资源的状态与新创建的托管资源同步。

#### 查看发现的资源

crossplane 会发现受管资源并填充{{<hover label="ooPopulated" line="12">}}status.atProvider{{</hover>}}字段中填入外部资源的 Values 值。

```yaml {label="ooPopulated",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
  forProvider:
    region: us-central1
status:
  atProvider:
    connectionName: crossplane-playground:us-central1:my-external-database
    databaseVersion: POSTGRES_14
    deletionProtection: true
    firstIpAddress: 35.184.74.79
    id: my-external-database
    publicIpAddress: 35.184.74.79
    region: us-central1
    # Removed for brevity
    settings:
    - activationPolicy: ALWAYS
      availabilityType: REGIONAL
      diskSize: 100
      # Removed for brevity
      pricingPlan: PER_USE
      tier: db-custom-4-26624
      version: 4
  conditions:
  - lastTransitionTime: "2023-02-22T07:16:51Z"
    reason: Available
    status: "True"
    type: Ready
  - lastTransitionTime: "2023-02-22T07:16:51Z"
    reason: ReconcileSuccess
    status: "True"
    type: Synced
```

<!-- vale off -->

## 控制导入的 ObserveOnly 资源

<!-- vale on -->

通过在导入后更改 "managementPolicies"，crossplane 可以主动控制只观察导入的资源。

更改 {{<hover label="fc" line="8">}}管理策略{{</hover>}}字段改为{{<hover label="fc" line="8">}}["*"]{{</hover>}}.

从{{<hover label="fc" line="16">}}status.atProvider{{</hover>}}中的参数值，并在 {{<hover label="fc" line="9">}}spec.forProvider{{</hover>}}.

{{< hint "tip" >}}手动将重要的 `spec.atProvider` Values 复制到 `spec.forProvider`。{{< /hint >}}

```yaml {label="fc"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["*"]
  forProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    settings:
    - diskSize: 100
      tier: db-custom-4-26624
status:
  atProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    # Removed for brevity
    settings:
    - diskSize: 100
      tier: db-custom-4-26624
      # Removed for brevity
  conditions:
    - lastTransitionTime: "2023-02-22T07:16:51Z"
      reason: Available
      status: "True"
      type: Ready
    - lastTransitionTime: "2023-02-22T11:16:45Z"
      reason: ReconcileSuccess
      status: "True"
      type: Synced
```

Crossplane 现在可以完全管理导入的资源。 Crossplane 会在 Provider 的外部资源中应用对所管理资源的任何更改。
