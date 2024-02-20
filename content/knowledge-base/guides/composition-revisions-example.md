---

title: Composition Revision示例
状态:  beta
alphaVersion: "1.4"
betaVersion: "1.11"

---

本教程讨论 CompositionRevisions 如何工作以及如何管理复合资源（XR）更新。 本教程从定义了 "MyVPC "资源的 "Composition "和 "CompositeResourceDefinition"（XRD）开始，然后创建多个 XR 以观察不同的升级路径。 每次更新 Composition 时，crossplane 都会为创建的复合资源分配不同的 CompositionRevisions。

## 准备工作

### 安装 crossplane

安装 Crossplane v1.11.0 或更高版本，并等待 Crossplane pod 运行。

```shell
kubectl create namespace crossplane-system
helm repo add crossplane-master https://charts.crossplane.io/master/
helm repo update
helm install crossplane --namespace crossplane-system crossplane-master/crossplane --devel --version 1.11.0-rc.0.108.g0521c32e
kubectl get pods -n crossplane-system
```

预期产出: 

```shell
NAME READY STATUS RESTARTS AGE
crossplane-7f75ddcc46-f4d2z 1/1 Running 0 9s
crossplane-rbac-manager-78bd597746-sdv6w 1/1 Running 0 9s
```

### 部署composition和 XRD 示例

应用示例 Composition。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  labels:
    channel: dev
  name: myvpcs.aws.example.upbound.io
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: aws.example.upbound.io/v1alpha1
    kind: MyVPC
  resources:
  - base:
      apiVersion: ec2.aws.upbound.io/v1beta1
      kind: VPC
      spec:
        forProvider:
          region: us-west-1
          cidrBlock: 192.168.0.0/16
          enableDnsSupport: true
          enableDnsHostnames: true
    name: my-vcp
```

应用 XRD 示例。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: myvpcs.aws.example.upbound.io
spec:
  group: aws.example.upbound.io
  names:
    kind: MyVPC
    plural: myvpcs
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
            properties:
              id:
                type: string 
                description: ID of this VPC that other objects will use to refer to it. 
            required:
            - id
```

验证 Crossplane 是否创建了 Composition 修订版

```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```

预期产出: 

```shell
NAME REVISION CHANNEL
myvpcs.aws.example.upbound.io-ad265bc 1 dev
```

{{< hint "note" >}}标签 `dev` 会自动从 Composition 中创建。{{< /hint >}}

## 创建 Composition 资源

本教程有四个复合资源，涵盖不同的更新策略和composition物选择选项。 默认行为是将 XR 更新到composition物的最新修订版。 不过，可以通过在 XR 中设置 `compositionUpdatePolicy: Manual` 来更改。也可以使用 `compositionRevisionSelector.matchLabels` 和 `compositionUpdatePolicy: Automatic` 来选择带有特定标签的最新修订版。

### 默认更新策略

创建的 XR 未定义 "compositionUpdatePolicy"（组件更新策略）。 更新策略默认为 "Automatic"（自动）: 

```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-auto
spec:
  id: vpc-auto
```

预期产出: 

```shell
myvpc.aws.example.upbound.io/vpc-auto created
```

### 手动更新政策

使用 `compositionUpdatePolicy: Manual` 和 `compositionRevisionRef` 创建 Composition 资源。

```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-man
spec:
  id: vpc-man
  compositionUpdatePolicy: Manual
  compositionRevisionRef:
    name: myvpcs.aws.example.upbound.io-ad265bc
```

预期产出: 

```shell
myvpc.aws.example.upbound.io/vpc-man created
```

### 使用选择器

创建一个 XR，其 `compositionRevisionSelector` 为 `channel: dev`: 

```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind:  MyVPC
metadata:
  name: vpc-dev
spec:
  id: vpc-dev
  compositionRevisionSelector:
    matchLabels:
      channel: dev
```

预期产出: 

```shell
myvpc.aws.example.upbound.io/vpc-dev created
```

创建一个 XR，其 `compositionRevisionSelector` 为 `channel: staging`: 

```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-staging
spec:
  id: vpc-staging
  compositionRevisionSelector:
    matchLabels:
      channel: staging
```

预期产出: 

```shell
myvpc.aws.example.upbound.io/vpc-staging created
```

验证标签为 "channel: staging "的composition资源没有 "REVISION"。 所有其他 XRs 的 "REVISION "都与创建的composition修订版相匹配。

```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```

预期产出: 

```shell
NAME SYNCED REVISION POLICY MATCHLABEL
vpc-auto True myvpcs.aws.example.upbound.io-ad265bc Automatic   <none>
vpc-dev True myvpcs.aws.example.upbound.io-ad265bc Automatic map[channel:dev]
vpc-man True myvpcs.aws.example.upbound.io-ad265bc Manual      <none>
vpc-staging False    <none>                                  Automatic map[channel:staging]
```

{{< hint "note" >}}vpc-staging` XR 标签与任何现有的 Composition Revisions 不匹配。{{< /hint >}}

## 创建新的 Composition Revisions

当创建或更新 Composition 时，crossplane 会创建一个新的 CompositionRevision。 标签和 Annotations 的更改也会触发一个新的 CompositionRevision。

### 更新 Composition 标签

将 "Composition "标签更新为 "channel: staging": 

```shell
kubectl label composition myvpcs.aws.example.upbound.io channel=staging --overwrite
```

预期产出: 

```shell
composition.apiextensions.crossplane.io/myvpcs.aws.example.upbound.io labeled
```

验证 Crossplane 是否创建了新的 Composition 修订版本: 

```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```

预期产出: 

```shell
NAME REVISION CHANNEL
myvpcs.aws.example.upbound.io-727b3c8 2 staging
myvpcs.aws.example.upbound.io-ad265bc 1 dev
```

验证 crossplane 是否将 Composition 资源 `vpc-auto` 和 `vpc-staging` 分配给 Composition revision:2，XRs `vpc-man` 和 `vpc-dev` 仍分配给原来的 revision:1: 

```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```

预期产出: 

```shell
NAME SYNCED REVISION POLICY MATCHLABEL
vpc-auto True myvpcs.aws.example.upbound.io-727b3c8 Automatic   <none>
vpc-dev True myvpcs.aws.example.upbound.io-ad265bc Automatic map[channel:dev]
vpc-man True myvpcs.aws.example.upbound.io-ad265bc Manual      <none>
vpc-staging True myvpcs.aws.example.upbound.io-727b3c8 Automatic map[channel:staging]
```

{{< hint "note" >}}`vpc-auto` 总是使用最新的 Revision。 `vpc-staging` 现在与被引用到 Revision revision:2 的标签相匹配。{{< /hint >}}

### 更新成分规格和标签

更新 Composition 以禁用 VPC 中的 DNS 支持，并将标签从 "staging "改回 "dev"。

应用以下更改更新 "Composition "规格和标签: 

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  labels:
    channel: dev
  name: myvpcs.aws.example.upbound.io
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: aws.example.upbound.io/v1alpha1
    kind: MyVPC
  resources:
  - base:
      apiVersion: ec2.aws.upbound.io/v1beta1
      kind: VPC
      spec:
        forProvider:
          region: us-west-1
          cidrBlock: 192.168.0.0/16
          enableDnsSupport: false
          enableDnsHostnames: true
    name: my-vcp
```

预期产出: 

```shell
composition.apiextensions.crossplane.io/myvpcs.aws.example.upbound.io configured
```

验证 Crossplane 是否创建了新的 Composition 修订版本: 

```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```

预期产出: 

```shell
NAME REVISION CHANNEL
myvpcs.aws.example.upbound.io-727b3c8 2 staging
myvpcs.aws.example.upbound.io-ad265bc 1 dev
myvpcs.aws.example.upbound.io-f81c553 3 dev
```

{{< hint "note" >}}同时更改标签和规范值对于将新更改部署到 `dev` 频道至关重要。{{< /hint >}}

验证 crossplane 是否将 Composition 资源 `vpc-auto` 和 `vpc-dev` 分配给 Composition 修订版本:3。`vpc-staging` 分配给修订版本:2，`vpc-man` 仍分配给原来的修订版本:1: 

```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```

预期产出: 

```shell
NAME SYNCED REVISION POLICY MATCHLABEL
vpc-auto True myvpcs.aws.example.upbound.io-f81c553 Automatic   <none>
vpc-dev True myvpcs.aws.example.upbound.io-f81c553 Automatic map[channel:dev]
vpc-man True myvpcs.aws.example.upbound.io-ad265bc Manual      <none>
vpc-staging True myvpcs.aws.example.upbound.io-727b3c8 Automatic map[channel:staging]
```

{{< hint "note" >}}`vpc-dev` 与应用于修订版本 3 的更新标签相匹配。 `vpc-staging` 与应用于修订版本 2 的标签相匹配。{{< /hint >}}
