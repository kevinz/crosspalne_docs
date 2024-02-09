---

title: AWS 快速入门
weight: 100

---

将 crossplane 连接到 AWS，利用 [Upbound AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-family-aws/v0.37.0) 从 Kubernetes 创建和管理云资源。

本指南分为两部分: 

* 第 1 部分将介绍如何安装 crossplane、配置 Provider 以实现以下功能

这表明 crossplane 可以与 AWS 通信。

* [第二部分]({{< ref "provider-aws-part-2" >}}) 展示了如何使用 crossplane 构建和访问自定义 API。

## 先决条件

本快速入门需要

* 至少有 2 GB 内存的 Kubernetes 集群
* 在 Kubernetes 集群中创建 pod 和 secrets 的权限
* 版本为 3.2.0 或更高的[helm](https://helm.sh/)
* 具有创建 S3 存储桶权限的 AWS 账户
* AWS [访问密钥](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## 安装 AWS Providers

用 Kubernetes 配置文件将 AWS S3 Provider 安装到 Kubernetes 集群中。

```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.47.0
EOF
```

The Crossplane {{< hover label="provider" line="3" >}}Provider{{</hover>}}这些 CRD 允许你直接在 Kubernetes 中创建 AWS 资源。

使用 `kubectl get providers` 验证已安装的 Provider。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-aws-s3 True True xpkg.upbound.io/upbound/provider-aws-s3:v0.47.0 97s
upbound-provider-family-aws True True xpkg.upbound.io/upbound/provider-family-aws:v0.47.0 88s
```

S3 Provider 会安装第二个 Provider，即{{<hover label="getProvider" line="4">}}upbound-provider-family-aws。{{</hover >}}该家族提供程序管理所有 AWS 家族提供程序对 AWS 的身份验证。

您可以使用 `kubectl get crds` 查看新的 CRD。 每个 CRD 都映射到 crossplane 可以调配和管理的唯一 AWS 服务。

{{< hint type="tip" >}}请查看 [Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v0.47.0) 中所有支持的 CRD 的详细信息。{{< /hint >}}

## 为 AWS 创建 Kubernetes secrets

Provider 需要凭据来创建和管理 AWS 资源。 Provider 使用 Kubernetes _Secret_ 将凭据连接到 Provider。

从 AWS 密钥对中生成 Kubernetes _Secret_，然后配置 Provider 以使用它。

#### 生成 AWS 密钥对文件

对于基本用户身份验证，请使用 AWS Access keys 密钥对文件。

{{< hint type="tip" >}}AWS 文档](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) 提供了有关如何生成 AWS 访问密钥的信息。{{< /hint >}}

创建包含 AWS 账户 `aws_access_key_id` 和 `aws_secret_access_key` 的文本文件。

{{< editCode >}}

```ini {copy-lines="all"}
[default]
aws_access_key_id = $@<aws_access_key>$@
aws_secret_access_key = $@<aws_secret_key>$@
```

{{< /editCode >}}

将此文本文件保存为 `aws-credentials.txt`.

{{< hint type="note" >}}AWS Provider 文档的[身份验证](https://docs.upbound.io/providers/provider-aws/authentication/) 部分介绍了其他身份验证方法。{{< /hint >}}

### 使用 AWS 凭据创建 Kubernetes secret

Kubernetes 通用 secret 有名称和内容。 使用{{< hover label="kube-create-secret" line="1">}}kubectl create secret{{</hover >}}生成名为{{< hover label="kube-create-secret" line="2">}}aws-secret{{< /hover >}}的 {{< hover label="kube-create-secret" line="3">}}名称空间中名为{{</ hover >}}namespace 中名为 aws-secret 的秘密对象。

被引用时使用 {{< hover label="kube-create-secret" line="4">}}--从文件{{</hover>}}参数将值设置为  {{< hover label="kube-create-secret" line="4">}}aws-credentials.txt{{< /hover >}}文件的内容。

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic aws-secret \
-n crossplane-system \
--from-file=creds=./aws-credentials.txt
```

使用 `kubectl describe secret` 查看秘密

{{< hint type="note" >}}如果文本文件中有额外的空白，大小可能会更大。{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret aws-secret -n crossplane-system
Name:         aws-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  114 bytes
```

## 创建一个 ProviderConfig

A {{< hover label="providerconfig" line="3">}}ProviderConfig{{</ hover >}}自定义 AWS Provider 的设置。

应用{{< hover label="providerconfig" line="3">}}ProviderConfig{{</ hover >}}与该 Kubernetes 配置文件一起使用: 

```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```

这会将保存为 Kubernetes secret 的 AWS 凭据作为{{< hover label="providerconfig" line="9">}}secretRef{{</ hover>}}.

规格.凭据.secretRef.名称{{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.name{{< /hover >}}值是包含 AWS 凭据的 Kubernetes secret 的名称，在{{< hover label="providerconfig" line="10">}}spec.credentials.secretRef.namespace 中包含 AWS 凭据的 Kubernetes 密钥名称。{{< /hover >}}.

## 创建托管资源

受管资源是指 Crossplane 在 Kubernetes 集群之外创建和管理的任何资源。

本指南使用 crossplane 创建 AWS S3 存储桶。

S3 存储桶是一种_受管资源_。

{{< hint type="note" >}}AWS S3 存储桶名称必须是全局唯一的。 为生成唯一名称，示例使用了随机散列。 任何唯一名称都可以接受。{{< /hint >}}

```yaml {label="xr"}
cat <<EOF | kubectl create -f -
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  generateName: crossplane-bucket-
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
EOF
```

版本 {{< hover label="xr" line="3">}}apiVersion{{< /hover >}}和{{< hover label="xr" line="4">}}类型{{</hover >}}来自 Provider 的 CRD。

元数据 {{< hover label="xr" line="6">}}metadata.name{{< /hover >}}<hash>值是在 AWS 中创建的 S3 存储桶的名称。 本示例在{{< hover label="xr" line="6">}}$bucket{{</hover >}}变量中使用生成的名称 "crossplane-bucket- "。

规格 {{< hover label="xr" line="9">}}spec.forProvider.region{{< /hover >}}告诉 AWS 在部署资源时要使用哪个 AWS 区域。

区域可以是任何 [AWS 区域端点](https://docs.aws.amazon.com/general/latest/gr/rande.html#regional-endpoints) 代码。

使用 `kubectl get buckets` 来验证 crossplane 是否创建了水桶。

{{< hint type="tip" >}}当 `READY` 和 `SYNCED` 值为 `True` 时，crossplane 会创建存储桶。 这可能需要长达 5 分钟的时间。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get buckets
NAME READY SYNCED EXTERNAL-NAME AGE
crossplane-bucket-hhdzh True True crossplane-bucket-hhdzh 5s
```

## 删除托管资源

在关闭 Kubernetes 集群之前，删除刚刚创建的 S3 存储桶。

使用 `kubectl delete bucket<bucketname>` 删除水桶。

```shell {copy-lines="1"}
kubectl delete bucket crossplane-bucket-hhdzh
bucket.s3.aws.upbound.io "crossplane-bucket-hhdzh" deleted
```

## 下一步

* [**继续第二部分**]({{< ref "provider-aws-part-2">}})来创建和被 crossplane 引用的自定义 API。
* 在[Provider CRD reference](https://marketplace.upbound.io/providers/upbound/provider-family-aws/)中探索 Crossplane 可以配置的 AWS 资源。
* 加入[Crossplane Slack](https://slack.crossplane.io/)，与Crossplane用户和贡献者建立联系。
