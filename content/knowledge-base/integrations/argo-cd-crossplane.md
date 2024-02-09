---

title: 使用 Argo CD 配置 crossplane
weight: 270

---

[Argo CD](https://argoproj.github.io/cd/)和[Crossplane](https://crossplane.io)是一个很好的组合。Argo CD 提供 GitOps 功能，而 Crossplane 可将任何 Kubernetes 集群转化为所有资源的通用控制平面（Universal Control Plane）。 要使二者正常协同工作，需要一些配置细节。 本文档将帮助您了解这些要求。 建议将 Argo CD 2.4.8 或更高版本与 Crossplane 结合使用。

Argo CD 会将存储在 Git 仓库中的 Kubernetes 资源清单与运行在 Kubernetes 集群（GitOps）中的资源清单同步。 配置 Argo CD 跟踪资源的方式有多种。 使用 crossplane 时，需要将 Argo CD 配置为使用基于 Annotations 的资源跟踪。更多详情请参见 [Argo CD docs](https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/)。

### 使用 crossplane 配置 Argo CD

#### 设置资源跟踪方法

为了使 Argo CD 能够正确跟踪包含 crossplane 相关对象的应用程序资源，需要将其配置为使用 Annotations 机制。

要对其进行配置，请在 `argocd` `Namespace` 中编辑 `argocd-cm``ConfigMap` 如下: 

```yaml
apiVersion: v1
kind: ConfigMap
data:
  application.resourceTrackingMethod: annotation
```

#### 设置健康状况

Argo CD 内置了 Kubernetes 资源的健康评估功能。 社区直接在 Argo 的 [repository](https://github.com/argoproj/argo-cd/tree/master/resource_customizations)中支持某些检查。例如，"pkg.crossplane.io "中的 "Provider "已经声明式，这意味着无需进一步配置。

Argo CD 还能在每个实例中自定义这些检查，这就是用来为 Provider 的 CRD 提供支持的机制。

要对其进行配置，请编辑 `argocd` `Namespace` 中的 `argocd-cm``ConfigMap` 。{{<hint "note">}}{{<hover label="argocfg" line="22">}}提供程序配置{{</hover>}}可能没有状态或`status.users`字段。{{</hint>}}

```yaml {label="argocfg"}
apiVersion: v1
kind: ConfigMap
data:
  application.resourceTrackingMethod: annotation
  resource.customizations: |
    "*.upbound.io/*":
      health.lua: |
        health_status = {
          status = "Progressing",
          message = "Provisioning ..."
        }

        local function contains (table, val)
          for i, v in ipairs(table) do
            if v == val then
              return true
            end
          end
          return false
        end

        local has_no_status = {
          "ProviderConfig",
          "ProviderConfigUsage"
        }

        if obj.status == nil and contains(has_no_status, obj.kind) then
          health_status.status = "Healthy"
          health_status.message = "Resource is up-to-date."
          return health_status
        end

        if obj.status == nil or obj.status.conditions == nil then
          if obj.kind == "ProviderConfig" and obj.status.users ~= nil then
            health_status.status = "Healthy"
            health_status.message = "Resource is in use."
            return health_status
          end
          return health_status
        end

        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "LastAsyncOperation" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Synced" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Ready" then
            if condition.status == "True" then
              health_status.status = "Healthy"
              health_status.message = "Resource is up-to-date."
              return health_status
            end
          end
        end

        return health_status

    "*.crossplane.io/*":
      health.lua: |
        health_status = {
          status = "Progressing",
          message = "Provisioning ..."
        }

        local function contains (table, val)
          for i, v in ipairs(table) do
            if v == val then
              return true
            end
          end
          return false
        end

        local has_no_status = {
          "Composition",
          "CompositionRevision",
          "DeploymentRuntimeConfig",
          "ControllerConfig"
        }
        if obj.status == nil and contains(has_no_status, obj.kind) then
            health_status.status = "Healthy"
            health_status.message = "Resource is up-to-date."
          return health_status
        end

        if obj.status == nil or obj.status.conditions == nil then
          return health_status
        end

        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "LastAsyncOperation" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Synced" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if contains({"Ready", "Healthy", "Offered", "Established"}, condition.type) then
            if condition.status == "True" then
              health_status.status = "Healthy"
              health_status.message = "Resource is up-to-date."
              return health_status
            end
          end
        end

        return health_status
```

#### 设置资源排除

Crossplane 提供商会为其处理的每个托管资源（MR）生成一个 "ProviderConfigUsage"。 该资源用于表示 MR 与 ProviderConfig 之间的关系，以便控制器在删除 ProviderConfig 时将其用作 Finalizer。 Crossplane 的最终用户不应与该资源交互。

随着资源和类型数量的增加，Argo CD UI 的反应能力也会受到影响。 为减少资源数量，我们建议在 Argo CD UI 中隐藏所有 `ProviderConfigUsage` 资源。

要配置资源排除，请按如下方式编辑 `argocd` `Namespace` 中的 `argocd-cm``ConfigMap` : 

```yaml
apiVersion: v1
kind: ConfigMap
data:
    resource.exclusions: |
      - apiGroups:
        - "*"
        kinds:
        - ProviderConfigUsage
```

使用 `"*"` 作为 apiGroups 将使该机制适用于所有 crossplane Providerers。

#### 提高 k8s 客户端 QPS

随着控制平面上 CRD 数量的增加，Argo CD 应用程序控制器需要向 Kubernetes API 发送的查询量也会增加。 如果出现这种情况，您可以增加 Argo CD Kubernetes 客户端的速率限制。

将环境变量 `ARGOCD_K8S_CLIENT_QPS` 设为 `300`，以提高与大量 CRD 的兼容性。

ARGOCD_K8S_CLIENT_QPS` 的默认值是 50，修改该值也会更新 `ARGOCD_K8S_CLIENT_BURST`，因为它的默认值是 `ARGOCD_K8S_CLIENT_QPS` x 2。