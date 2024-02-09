---

title: Azure 快速入门 
weight: 110

---

将 crossplane 连接到 Azure，利用 [Upbound Azure Provider](https://marketplace.upbound.io/providers/upbound/provider-family-azure/) 从 Kubernetes 创建和管理云资源。

本指南分为两部分: 

* 第 1 部分将介绍如何安装 crossplane、配置 Provider 以实现以下功能

这表明 crossplane 可以与 Azure 通信。

* [第二部分]({{< ref "provider-azure-part-2" >}}) 展示了如何使用 crossplane 构建和访问自定义 API。

## 先决条件

本快速入门需要

* 至少有 2 GB 内存的 Kubernetes 集群
* 在 Kubernetes 集群中创建 pod 和 secrets 的权限
* 版本为 3.2.0 或更高的[Helm](https://helm.sh/)
* 具有创建 [Azure 虚拟机](https://learn.microsoft.com/en-us/azure/virtual-machines/) 和 [虚拟网络](https://learn.microsoft.com/en-us/azure/virtual-network/) 权限的 Azure 账户
* 具有创建 Azure [服务委托人](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals#service-principal-object) 和 [Azure 资源组](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal) 权限的 Azure 帐户

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## 安装 Azure Provider

使用 Kubernetes 配置文件将 Azure Network 资源 Provider 安装到 Kubernetes 集群中。

```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-network
spec:
  package: xpkg.upbound.io/upbound/provider-azure-network:v0.34.0
EOF
```

The Crossplane {{< hover label="provider" line="3" >}}Provider{{</hover>}}这些 CRD 可让您直接在 Kubernetes 中创建 Azure 资源。

使用 `kubectl get providers` 验证已安装的 Provider。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-azure-network True True xpkg.upbound.io/upbound/provider-azure-network:v0.34.0 38s
upbound-provider-family-azure True True xpkg.upbound.io/upbound/provider-family-azure:v0.34.0 26s
```

网络提供商会安装第二个提供商，即{{<hover label="getProvider" line="4">}}提供程序。{{</hover>}}家族提供程序管理所有 Azure 家族提供程序对 Azure 的身份验证。

您可以使用 `kubectl get crds` 查看新的 CRD。 每个 CRD 都映射到 crossplane 可以提供和管理的唯一 Azure 服务。

{{< hint type="tip" >}}请查看 [Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-family-azure/v0.34.0) 中所有支持的 CRD 的详细信息。{{< /hint >}}

## 为 Azure 创建 Kubernetes secrets

Provider 需要凭据才能创建和管理 Azure 资源。 Provider 使用 Kubernetes _Secret_ 将凭据连接到 Provider。

本指南生成 Azure 服务 principal JSON 文件，并将其保存为 Kubernetes _Secret_ 文件。

#### 安装 Azure 命令行

生成[身份验证文件](https://docs.microsoft.com/en-us/azure/developer/go/azure-sdk-authorization#use-file-based-authentication) 需要使用 Azure 命令行。请按照微软提供的文档[下载并安装 Azure 命令行](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)。

登录 Azure 命令行。

```command
az login
```

#### 创建 Azure 服务委托人

按照 Azure 文档从 Azure 门户[查找您的订阅 ID](https://docs.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id)。

使用 Azure 命令行并被引用的订阅 ID 创建服务委托和身份验证文件。

{{< editCode >}}

```console {copy-lines="all"}
az ad sp create-for-rbac \
--sdk-auth \
--role Owner \
--scopes /subscriptions/$@<subscription_id>$@
```

{{< /editCode >}}

将您的 Azure JSON 输出保存为 `azure-credentials.json`。

{{< hint type="note" >}}Azure Provider [身份验证文档](https://github.com/upbound/provider-azure/blob/main/AUTHENTICATION.md) 介绍了其他身份验证方法。{{< /hint >}}

### 使用 Azure 凭据创建 Kubernetes secret

Kubernetes 通用 secret 有名称和内容。 使用 {{< hover label="kube-create-secret" line="1">}}kubectl create secret{{< /hover >}}生成名为 {{< hover label="kube-create-secret" line="2">}}azure-secret{{< /hover >}}的 {{< hover label="kube-create-secret" line="3">}}crossplane-system{{</ hover >}}namespace 中名为 azure-secret 的秘密对象。

<!-- vale gitlab.Substitutions = NO -->

<!-- ignore .json file name -->

被引用时使用 {{< hover label="kube-create-secret" line="4">}}--从文件{{</hover>}}参数将值设置为  {{< hover label="kube-create-secret" line="4">}}azure-credentials.json{{< /hover >}}文件的内容。

<!-- vale gitlab.Substitutions = YES -->

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic azure-secret \
-n crossplane-system \
--from-file=creds=./azure-credentials.json
```

使用 `kubectl describe secret` 查看秘密

{{< hint type="note" >}}如果文本文件中有额外的空白，大小可能会更大。{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret azure-secret -n crossplane-system
Name:         azure-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  629 bytes
```

## 创建一个 ProviderConfig

ProviderConfig "可自定义 Azure Provider 的设置。

应用 {{< hover label="providerconfig" line="5">}}提供商配置{{</ hover >}}命令应用 ProviderConfig: 

```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
metadata:
  name: default
kind: ProviderConfig
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-secret
      key: creds
EOF
```

这会将保存为 Kubernetes secret 的 Azure 凭据作为 {{< hover label="providerconfig" line="9">}}secretRef。{{</ hover>}}.

规格.凭据.secretRef.名称 {{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.name{{< /hover >}}值是包含 Azure 凭据的 Kubernetes secret 的名称，在 {{< hover label="providerconfig" line="10">}}spec.credentials.secretRef.namespace 中包含 Azure 凭据的 Kubernetes 密钥的名称。{{< /hover >}}.

## 创建托管资源

受管资源是指 Crossplane 在 Kubernetes 集群之外创建和管理的任何资源。 本示例使用 Crossplane 创建了一个 Azure 虚拟网络。 虚拟网络是受管资源。

{{< hint type="note" >}}添加您的 Azure 资源组名称。如果没有资源组，请按照 Azure 文档[创建资源组](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)。{{< /hint >}}

{{< editCode >}}

```yaml {label="xr"}
cat <<EOF | kubectl create -f -
apiVersion: network.azure.upbound.io/v1beta1
kind: VirtualNetwork
metadata:
  name: crossplane-quickstart-network
spec:
  forProvider:
    addressSpace:
      - 10.0.0.0/16
    location: "Sweden Central"
    resourceGroupName: docs
EOF
```

{{< /editCode >}}

版本 {{< hover label="xr" line="2">}}apiVersion{{< /hover >}}和{{< hover label="xr" line="3">}}类型{{</hover >}}来自 Provider 的 CRD。

规格 {{< hover label="xr" line="10">}}spec.forProvider.location{{< /hover >}}会告诉 Azure 在部署资源时要使用哪个位置。

使用 `kubectl get virtualnetwork.network` 验证 crossplane 是否创建了 Azure 虚拟网络。

{{< hint type="tip" >}}当 `READY` 和 `SYNCED` 值为 `True` 时，crossplane 会创建虚拟网络。 这可能需要 5 分钟。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get virtualnetwork.network
NAME READY SYNCED EXTERNAL-NAME AGE
crossplane-quickstart-network True True crossplane-quickstart-network 10m
```

## 删除托管资源

在关闭 Kubernetes 集群之前，删除刚刚创建的虚拟网络。

被引用 `kubectl delete virtualnetwork.network` 删除虚拟网络。

```shell {copy-lines="1"}
kubectl delete virtualnetwork.network crossplane-quickstart-network
virtualnetwork.network.azure.upbound.io "crossplane-quickstart-network" deleted
```

## 下一步

* [**继续第二部分**]({{< ref "provider-azure-part-2">}})来创建和被 crossplane 引用的自定义 API。
* 在[Provider CRD reference](https://marketplace.upbound.io/providers/upbound/provider-family-azure/)中探索 Crossplane 可以配置的 Azure 资源。
* 加入[Crossplane Slack](https://slack.crossplane.io/)，与Crossplane用户和贡献者建立联系。
