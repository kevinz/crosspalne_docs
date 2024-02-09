---

title: GCP 快速入门
weight: 140

---

将 crossplane 连接到 GCP，利用 [Upbound GCP Provider](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/) 从 Kubernetes 创建和管理云资源。

本指南分为两部分: 

* 第 1 部分将介绍如何安装 crossplane、配置 Provider 以实现以下功能

这表明 crossplane 可以与 GCP 通信。

* [第二部分]({{< ref "provider-gcp-part-2" >}}) 展示了如何使用 crossplane 构建和访问自定义 API。

## 先决条件

本快速入门需要

* 至少有 2 GB 内存的 Kubernetes 集群
* 在 Kubernetes 集群中创建 pod 和 secrets 的权限
* [Helm](https://helm.sh/)版本 3.2.0 或更高
* 具有创建存储桶权限的 GCP 账户
* GCP [账户密钥](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
* GCP [项目 ID](https://support.google.com/googleapi/answer/7014113?hl=en)

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## 安装 GCP Providers

使用 Kubernetes 配置文件将 Provider 安装到 Kubernetes 集群中。

```shell {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0
EOF
```

The Crossplane {{< hover label="provider" line="3" >}}Provider{{</hover>}}这些 CRD 可让您直接在 Kubernetes 中创建 GCP 资源。

使用 `kubectl get providers` 验证已安装的 Provider。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-gcp-storage True True xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0 36s
upbound-provider-family-gcp True True xpkg.upbound.io/upbound/provider-family-gcp:v0.41.0 29s
```

存储提供程序会安装第二个 Provider，即{{<hover label="getProvider" line="4">}}提供程序。{{</hover>}}该家族提供程序管理所有 GCP 家族提供程序对 GCP 的身份验证。

你可以使用 `kubectl get crds` 查看新的 CRD。 每个 CRD 都映射到 crossplane 可以提供和管理的唯一 GCP 服务。

{{< hint "tip" >}}请查看 [Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/) 中所有支持的 CRD 的详细信息。{{< /hint >}}

## 为 GCP 创建 Kubernetes secrets

Provider 需要凭据才能创建和管理 GCP 资源。 Provider 使用 Kubernetes _Secret_ 将凭据连接到 Provider。

首先从 Google 云服务账户 JSON 文件中生成 Kubernetes _Secret_，然后配置 Providers 以使用它。

### 生成 GCP 服务账户 JSON 文件

对于基本用户身份验证，请使用 Google 云服务账户 JSON 文件。

{{< hint "tip" >}}[GCP 文档](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) 提供了如何生成服务帐户 JSON 文件的信息。{{< /hint >}}

将此 JSON 文件保存为 `gcp-credentials.json

### 使用 GCP 凭据创建 Kubernetes secret

Kubernetes 通用 secret 有名称和内容。 使用{{< hover label="kube-create-secret" line="1">}}kubectl create secret{{< /hover >}}生成名为{{< hover label="kube-create-secret" line="2">}}gcp-secret{{< /hover >}}的{{< hover label="kube-create-secret" line="3">}}名称空间中名为{{</ hover >}}名称空间中生成名为 gcp-secret 的秘密对象。 {{< hover label="kube-create-secret" line="4">}}--从文件{{</hover>}}参数将值设置为{{< hover label="kube-create-secret" line="4">}}gcp-credentials.json{{< /hover >}}文件的内容。

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./gcp-credentials.json
```

使用 `kubectl describe secret` 查看秘密

{{< hint "note" >}}文件大小可能因内容而异。{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret gcp-secret -n crossplane-system
Name:         gcp-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  2330 bytes
```

## 创建一个 ProviderConfig

ProviderConfig "自定义 GCP Provider 的设置。

包括您的{{< hover label="providerconfig" line="7" >}}GCP 项目 ID{{< /hover >}}设置中。

{{< hint "tip" >}}从 `gcp-credentials.json` 文件中的 `project_id` 字段查找 GCP 项目 ID。{{< /hint >}}

应用{{< hover label="providerconfig" line="2">}}提供商配置{{</ hover >}}命令应用 ProviderConfig: 

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

这会将保存为 Kubernetes secret 的 GCP 凭据作为{{< hover label="providerconfig" line="10">}}secretRef{{</ hover>}}.

规格.凭据.secretRef.名称 {{< hover label="providerconfig" line="12">}}spec.credentials.secretRef.name{{< /hover >}}值是包含 GCP 凭据的 Kubernetes secret 的名称，在{{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.namespace 中包含 GCP 凭据的 Kubernetes 密钥名称。{{< /hover >}}.

## 创建托管资源

受管资源是指 Crossplane 在 Kubernetes 集群之外创建和管理的任何资源。 本示例使用 Crossplane 创建了一个 GCP 存储桶。 存储桶是受管资源。

{{< hint "note" >}}要生成唯一名称，请引用{{<hover label="xr" line="5">}}生成名称{{</hover >}}而不是 `name`。{{< /hint >}}

使用以下命令创建 Bucket: 

```yaml {label="xr",copy-lines="all"}
cat <<EOF | kubectl create -f -
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  generateName: crossplane-bucket-
  labels:
    docs.crossplane.io/example: provider-gcp
spec:
  forProvider:
    location: US
  providerConfigRef:
    name: default
EOF
```

版本 {{< hover label="xr" line="2">}}apiVersion{{< /hover >}}和{{< hover label="xr" line="3">}}类型{{</hover >}}来自 Provider 的 CRD。

规格 {{< hover label="xr" line="10">}}spec.forProvider.location{{< /hover >}}会告诉 GCP 在部署资源时要使用哪个 GCP 区域。 对于一个{{<hover label="xr" line="3">}}资源桶{{</hover >}}区域可以是任何 [GCP 多区域位置](https://cloud.google.com/storage/docs/locations#location-mr)

使用 `kubectl get bucket` 来验证 crossplane 是否创建了水桶。

{{< hint type="tip" >}}当 `READY` 和 `SYNCED` 值为 `True` 时，crossplane 会创建存储桶。 这可能需要长达 5 分钟的时间。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get bucket
NAME READY SYNCED EXTERNAL-NAME AGE
crossplane-bucket-8b7gw True True crossplane-bucket-8b7gw 2m2s
```

## 删除托管资源

在关闭 Kubernetes 集群之前，删除刚刚创建的 GCP 存储桶。

被引用 `kubectl delete bucket` 删除水桶。

{{<hint "tip" >}}使用 `--selector` flag 可按标签而非名称删除。{{</hint>}}

```shell {copy-lines="1"}
kubectl delete bucket --selector docs.crossplane.io/example=provider-gcp
bucket.storage.gcp.upbound.io "crossplane-bucket-8b7gw" deleted
```

## 下一步

* [**继续第二部分**]({{< ref "provider-gcp-part-2">}}) 创建一个

crossplane _Composite Resource_ 和_Claim_。

* 探索可以在 crossplane 中配置的 GCP 资源

[Provider CRD reference](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/)。

* 加入 [Crossplane Slack](https://slack.crossplane.io/)，与

crossplane 用户和投稿人。