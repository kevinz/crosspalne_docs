---

title: Azure Quickstart Part 2 
weight: 120 
tocHidden: true 
aliases: 
  - /master/getting-started/provider-azure-part-3

---

{{< hint "important" >}}本指南是系列指南的第二部分。

[第 1 部分**]({{<ref "provider-azure" >}}包括安装 crossplane 并将 Kubernetes 集群连接到 Azure。

{{< /hint >}}

本指南将指导您使用 crossplane 构建和访问自定义 API。

## 先决条件

* 完成 [快速入门第 1 部分] ({{<ref "provider-azure" >}}) 将 Kubernetes 连接到 Azure。
* 一个具有创建 Azure 虚拟机、资源的权限的 Azure 帐户。

群组和虚拟网络。

{{<expand "Skip part 1 and just get started" >}}

1.添加 Crossplane Helm 软件源并安装 Crossplane

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

2.当 crossplane pod 完成安装并准备就绪时，应用 Azure Provider

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

3.使用 Azure CLI 创建服务委托，并将 JSON 输出保存为 `azure-crednetials.json

{{< editCode >}}

```console
az ad sp create-for-rbac \
--sdk-auth \
--role Owner \
--scopes /subscriptions/$@<subscription_id>$@
```

{{</ editCode >}}

4.从 Azure JSON 文件创建 Kubernetes secret。

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic azure-secret \
-n crossplane-system \
--from-file=creds=./azure-credentials.json
```

5.创建 _ProviderConfig_

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

{{</expand >}}

## 创建自定义应用程序接口

<!-- vale alex.Condescending = NO -->

Crossplane 允许您为用户构建自己的自定义 API，抽象掉有关云提供商及其资源的细节。 您可以随心所欲地让自己的 API 变得复杂或简单。

<!-- vale alex.Condescending = YES -->

下面是一个自定义 API 的示例。

```yaml {label="exAPI"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
metadata:
  name: my-vm
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

本指南被引用的组是{{<hover label="version" line="1">}}compute.example.com{{</hover>}}.

由于这是 API 的第一个版本，本指南被引用的版本为{{<hover label="version" line="1">}}v1alpha1{{</hover>}}.

```yaml {label="version",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
```

### 定义一种

API 组是相关 API 的逻辑集合。 在一个组中，有代表不同资源的单个种类。

例如，一个 "计算 "组可能有 "虚拟机 "和 "裸机 "两种类型。

种类 "可以是任何东西，但必须是[UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects)。

该应用程序接口的种类是{{<hover label="kind" line="2">}}虚拟机{{</hover>}}

```yaml {label="kind",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
```

### 定义规格

应用程序接口最重要的部分是模式。 模式定义了用户接受的输入。

该应用程序接口允许用户提供{{<hover label="spec" line="4">}}位置{{</hover>}}运行云资源的位置。

所有其他资源设置都不能由用户配置。 这使得 crossplane 可以执行任何策略和标准，而不用担心用户出错。

```yaml {label="spec",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
spec: 
  location: "US"
```

### 应用应用程序接口

crossplane 被引用{{<hover label="xrd" line="3">}}Composition 资源定义{{</hover>}}(也称为 "XRD"）在 Kubernetes 中安装您的自定义 API。

XRD {{<hover label="xrd" line="6">}}规范{{</hover>}}包含有关应用程序接口的所有信息，包括{{<hover label="xrd" line="7">}}组{{</hover>}},{{<hover label="xrd" line="12">}}版本{{</hover>}},{{<hover label="xrd" line="9">}}种类{{</hover>}}和{{<hover label="xrd" line="13">}}模式{{</hover>}}.

XRD 的 {{<hover label="xrd" line="5">}}名称{{</hover>}}必须是 {{<hover label="xrd" line="10">}}复数{{</hover>}}和{{<hover label="xrd" line="7">}}组{{</hover>}}.

模式 {{<hover label="xrd" line="13">}}模式{{</hover>}}被引用为{{<hover label="xrd" line="14">}}开放式应用程序接口 v3{{</hover>}}规范来定义应用程序接口 {{<hover label="xrd" line="17">}}规范{{</hover>}}.

应用程序接口定义了一个 {{<hover label="xrd" line="20">}}位置{{</hover>}}必须是 {{<hover label="xrd" line="22">}}之一{{</hover>}}或{{<hover label="xrd" line="23">}}欧盟{{</hover>}}或{{<hover label="xrd" line="24">}}美国{{</hover>}}.

应用此 XRD 在 Kubernetes 集群中创建自定义 API。

```yaml {label="xrd",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: virtualmachines.compute.example.com
spec:
  group: compute.example.com
  names:
    kind: VirtualMachine
    plural: virtualmachines
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
    kind: VirtualMachineClaim
    plural: virtualmachineclaims
EOF
```

添加 {{<hover label="xrd" line="29">}}索赔名称{{</hover>}}允许用户在集群级别使用{{<hover label="xrd" line="9">}}虚拟机{{</hover>}}端点访问该 API，或在名称空间中使用{{<hover label="xrd" line="30">}}虚拟机索赔{{</hover>}}端点访问该 API。

namespace 范围内的 API 是 crossplane _Claim_。

{{<hint "tip" >}}有关 Composition 资源定义的字段和选项的更多详情，请阅读 [XRD 文档]({{<ref "../concepts/composite-resource-definitions">}}).{{< /hint >}}

使用 `kubectl get xrd` 查看已安装的 XRD。

```shell {copy-lines="1"}
kubectl get xrd
NAME ESTABLISHED OFFERED AGE
virtualmachines.compute.example.com True True 43s
```

使用 `kubectl api-resources | grep VirtualMachine` 查看新的自定义 API 端点

```shell {copy-lines="1",label="apiRes"}
kubectl api-resources | grep VirtualMachine
virtualmachineclaims compute.example.com/v1alpha1 true VirtualMachineClaim
virtualmachines compute.example.com/v1alpha1 false VirtualMachine
```

## 创建部署模板

当用户访问自定义 API 时，crossplane 会接收他们的输入，并将其与描述要部署的基础架构的模板相结合。 Crossplane 将此模板称为_Composition_。

构成 {{<hover label="comp" line="3">}}Composition{{</hover>}}模板中的每个条目都是一个完整的资源定义，定义了所有资源设置和元数据，如标签和 Annotations。

此模板会创建一个 Azure{{<hover label="comp" line="11">}}虚拟机{{</hover>}}{{<hover label="comp" line="46">}}网络接口{{</hover>}},{{<hover label="comp" line="69">}}子网{{</hover>}}{{<hover label="comp" line="90">}}虚拟网络{{</hover>}}和{{<hover label="comp" line="110">}}资源组{{</hover>}}.

crossplane 被引用为 {{<hover label="comp" line="34">}}补丁{{</hover>}}将用户的输入应用到资源模板中。 该 Composition 将用户的{{<hover label="comp" line="36">}}位置{{</hover>}}输入，并将其作为{{<hover label="comp" line="37">}}位置{{</hover>}}被引用到单个资源中。

将此 Composition 应用于您的集群。

```yaml {label="comp",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: crossplane-quickstart-vm-with-network
spec:
  resources:
    - name: quickstart-vm
      base:
        apiVersion: compute.azure.upbound.io/v1beta1
        kind: LinuxVirtualMachine
        spec:
          forProvider:
            adminUsername: adminuser
            adminSshKey:
              - publicKey: ssh-rsa
                  AAAAB3NzaC1yc2EAAAADAQABAAABAQC+wWK73dCr+jgQOAxNsHAnNNNMEMWOHYEccp6wJm2gotpr9katuF/ZAdou5AaW1C61slRkHRkpRRX9FA9CYBiitZgvCCz+3nWNN7l/Up54Zps/pHWGZLHNJZRYyAB6j5yVLMVHIHriY49d/GZTZVNB8GoJv9Gakwc/fuEZYYl4YDFiGMBP///TzlI4jhiJzjKnEvqPFki5p2ZRJqcbCiF4pJrxUQR/RXqVFQdbRLZgYfJ8xGB878RENq3yQ39d8dVOkq4edbkzwcUmwwwkYVPIoDGsYLaRHnG+To7FvMeyO7xDVQkMKzopTQV8AuKpyvpqu0a9pWOMaiCyDytO7GGN
                  example@docs.crossplane.io
                username: adminuser
            location: "Central US"
            osDisk:
              - caching: ReadWrite
                storageAccountType: Standard_LRS
            resourceGroupNameSelector:
              matchControllerRef: true
            size: Standard_B1ms
            sourceImageReference:
              - offer: debian-11
                publisher: Debian
                sku: 11-backports-gen2
                version: latest
            networkInterfaceIdsSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: quickstart-nic
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: NetworkInterface
        spec:
          forProvider:
            ipConfiguration:
              - name: crossplane-quickstart-configuration
                privateIpAddressAllocation: Dynamic
                subnetIdSelector:
                  matchControllerRef: true
            location: "Central US"
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"            
    - name: quickstart-subnet
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: Subnet
        spec:
          forProvider:
            addressPrefixes:
              - 10.0.1.0/24
            virtualNetworkNameSelector:
              matchControllerRef: true
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: quickstart-network
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: VirtualNetwork
        spec:
          forProvider:
            addressSpace:
              - 10.0.0.0/16
            location: "Central US"
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: crossplane-resourcegroup
      base:
        apiVersion: azure.upbound.io/v1beta1
        kind: ResourceGroup
        spec:
          forProvider:
            location: Central US
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
  compositeTypeRef:
    apiVersion: compute.example.com/v1alpha1
    kind: VirtualMachine
EOF
```

复合类型 {{<hover label="comp" line="52">}}compositeTypeRef{{</hover >}}定义了哪些自定义 API 可以使用此模板创建资源。

{{<hint "tip" >}}请阅读 [Composition documentation]({{<ref "../concepts/compositions">}}) 获取更多关于配置 Composition 和所有可用选项的信息。

阅读[补丁和转换文档]({{<ref "../concepts/patch-and-transform">}})，了解有关 crossplane 如何使用补丁将用户输入映射到 Composition 资源模板的更多信息。{{< /hint >}}

使用 `kubectl get composition` 查看 Composition

```shell {copy-lines="1"}
kubectl get composition
NAME XR-KIND XR-APIVERSION AGE
crossplane-quickstart-vm-with-network XVirtualMachine custom-api.example.org/v1alpha1 77s
```

## 安装 Azure 虚拟机 Providers

第 1 部分只安装了 Azure 虚拟网络 Provider。 要部署虚拟机还需要 Azure 计算提供程序。

将新的 Provider 添加到集群中。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-compute
spec:
  package: xpkg.upbound.io/upbound/provider-azure-compute:v0.34.0
EOF
```

使用 `kubectl get Provider` 查看新的计算提供程序。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-azure-compute True True xpkg.upbound.io/upbound/provider-azure-compute:v0.34.0 25s
provider-azure-network True True xpkg.upbound.io/upbound/provider-azure-network:v0.34.0 3h
upbound-provider-family-azure True True xpkg.upbound.io/upbound/provider-family-azure:v0.34.0 3h
```

## 访问自定义应用程序接口

安装了自定义应用程序接口（XRD）并与资源模板（Composition）关联后，用户就可以访问应用程序接口来创建资源。

创建一个 {{<hover label="xr" line="3">}}虚拟机{{</hover>}}对象来创建云资源。

```yaml {copy-lines="all",label="xr"}
cat <<EOF | kubectl apply -f -
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
metadata:
  name: my-vm
spec: 
  location: "EU"
EOF
```

使用 `kubectl get VirtualMachine` 查看资源。

{{< hint "note" >}}资源提供可能需要 5 分钟。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get VirtualMachine
NAME SYNCED READY COMPOSITION AGE
my-vm True True crossplane-quickstart-vm-with-network 3m3s
```

该对象是一个 crossplane _composite resource_（也称为 `XR`）。 它是一个单独的对象，代表从 Composition 模板创建的资源集合。

使用 `kubectl get managed` 查看单个资源

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED EXTERNAL-NAME AGE
resourcegroup.azure.upbound.io/my-vm-7jb4n True True my-vm-7jb4n 3m43s

NAME READY SYNCED EXTERNAL-NAME AGE
linuxvirtualmachine.compute.azure.upbound.io/my-vm-5h7p4 True True my-vm-5h7p4 3m43s

NAME READY SYNCED EXTERNAL-NAME AGE
networkinterface.network.azure.upbound.io/my-vm-j7fpx True True my-vm-j7fpx 3m43s

NAME READY SYNCED EXTERNAL-NAME AGE
subnet.network.azure.upbound.io/my-vm-b2dqt True True my-vm-b2dqt 3m43s

NAME READY SYNCED EXTERNAL-NAME AGE
virtualnetwork.network.azure.upbound.io/my-vm-pd2sw True True my-vm-pd2sw 3m43s
```

访问应用程序接口可创建模板中定义的所有五个资源，并将它们链接在一起。

查看特定资源，看它是在 API 中被引用的位置创建的。

```yaml {copy-lines="1"}
kubectl describe linuxvirtualmachine | grep Location
    Location:                         Sweden Central
    Location:                         swedencentral
```

使用 `kubectl delete VirtualMachine` 删除资源。

```shell {copy-lines="1"}
kubectl delete VirtualMachine my-vm
virtualmachine.compute.example.com "my-vm" deleted
```

使用 `kubectl get managed` 验证 crossplane 是否删除了资源

{{<hint "note" >}}删除资源可能需要 5 分钟。{{< /hint >}}

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 使用带有 namespace 的应用程序接口

访问应用程序接口 `VirtualMachine` 是在集群范围内进行的。 大多数组织将用户隔离到 namespace 中。

Crossplane _Claim_ 是名称空间中的自定义应用程序接口。

创建 _Claim_ 就像访问自定义 API 端点一样，只不过是使用了{{<hover label="claim" line="3">}}类型{{</hover>}}从自定义 API 的 "索赔名称 "中创建。

创建一个新的命名空间，以测试在其中创建一个 claims。

```shell
kubectl create namespace crossplane-test
```

然后在 `crossplane-test` 名称空间中创建一个 claim。

```yaml {label="claim",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachineClaim
metadata:
  name: my-namespaced-vm
  namespace: crossplane-test
spec: 
  location: "EU"
EOF
```

使用 `kubectl get claim -n crossplane-test` 查看索赔。

```shell {copy-lines="1"}
kubectl get claim -n crossplane-test
NAME SYNCED READY CONNECTION-SECRET AGE
my-namespaced-vm True True 5m11s
```

claims 会自动创建一个复合资源，该资源会创建托管资源。

使用 `kubectl get composite` 查看 Crossplane 创建的 Composition 资源。

```shell {copy-lines="1"}
kubectl get composite
NAME SYNCED READY COMPOSITION AGE
my-namespaced-vm-r7gdr True True crossplane-quickstart-vm-with-network 5m33s
```

同样，使用 `kubectl get managed` 查看托管资源。

```shell {copy-lines="1"}
NAME READY SYNCED EXTERNAL-NAME AGE
resourcegroup.azure.upbound.io/my-namespaced-vm-r7gdr-cvzw6 True True my-namespaced-vm-r7gdr-cvzw6 5m51s

NAME READY SYNCED EXTERNAL-NAME AGE
linuxvirtualmachine.compute.azure.upbound.io/my-namespaced-vm-r7gdr-vrbgb True True my-namespaced-vm-r7gdr-vrbgb 5m51s

NAME READY SYNCED EXTERNAL-NAME AGE
networkinterface.network.azure.upbound.io/my-namespaced-vm-r7gdr-hwrb8 True True my-namespaced-vm-r7gdr-hwrb8 5m51s

NAME READY SYNCED EXTERNAL-NAME AGE
subnet.network.azure.upbound.io/my-namespaced-vm-r7gdr-gh468 True True my-namespaced-vm-r7gdr-gh468 5m51s

NAME READY SYNCED EXTERNAL-NAME AGE
virtualnetwork.network.azure.upbound.io/my-namespaced-vm-r7gdr-5qhz7 True True my-namespaced-vm-r7gdr-5qhz7 5m51s
```

删除 claim 会删除所有 crossplane 生成的资源。

`kubectl delete claim -n crossplane-test my-VirtualMachine-database`.

```shell {copy-lines="1"}
kubectl delete claim -n crossplane-test my-namespaced-vm
virtualmachineclaim.compute.example.com "my-namespaced-vm" deleted
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

* 在[Provider CRD reference](https://marketplace.upbound.io/providers/upbound/provider-family-azure/)中探索 Crossplane 可以配置的 Azure 资源。
* 加入 [Crossplane Slack](https://slack.crossplane.io/) 并与 Crossplane 用户和贡献者联系。
* 阅读更多有关[Crossplane 概念]({{<ref "../concepts">}})，了解 Crossplane 还能做些什么。
