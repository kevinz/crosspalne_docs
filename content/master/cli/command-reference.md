---

weight: 50
title: 命令参考
description: "Crossplane CLI 的命令参考"

---

<!-- vale Google.Headings = NO -->

`crossplane` CLI 提供了一些实用程序，让使用 crossplane 变得更容易。

请阅读 [Crossplane CLI overview]({{<ref "../cli">}}) 页面，了解有关安装 `crossplane` 的信息。

## global flags

以下 flag 适用于所有命令。

{{< table "table table-sm table-striped">}}| 短标志 | 长标志 | 说明 | |------------|-------------|------------------------------| | `-h` | `--help` | 显示上下文相关帮助。 | | `-v` | `--version` | 打印版本并退出。 | | | `--verbose` | 打印冗余输出。{{< /table >}}

## xpkg

crossplane xpkg` 命令用于创建、安装和更新 Crossplane [packages]({{<ref "../concepts/packages">}}) 以及启用身份验证和将 crossplane 软件包发布到 crossplane 软件包注册表。

### xpkg build

被引用的 "crossplane xpkg build "提供了构建 crossplane 软件包的自动化和简化。

crossplane CLI 将 YAML 文件目录组合在一起，并将其打包为 [OCI 容器镜像](https://opencontainers.org/)。

CLI 应用所需的 Annotations 和 Values，以满足 [crossplane XPKG 规范](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md)。

crossplane "CLI支持构建[配置]({{< ref "../concepts/packages" >}})、[函数]({{<ref "../concepts/composition-functions">}}) 和 [Provider]({{<ref "../concepts/providers" >}}) 包类型。

#### flag

{{< table "table table-sm table-striped">}}| Short flag   | Long flag                            | Description                    |
| ------------ | -------------                        | ------------------------------ |
|              | `--embed-runtime-image-name=NAME`    |  The image name and tag of an image to include in the package. Only for provider and function packages. |
|              | `--embed-runtime-image-tarball=PATH` |  The filename of an image to include in the package. Only for provider and function packages.                              |
| `-e`         | `--examples-root="./examples"`       |  The path to a directory of examples related to the package.                               |
|              | `--ignore=PATH,...`                  |  List of files and directories to ignore.                              |
| `-o`         | `--package-file=PATH`                |  Directory and filename of the created package.                             |
| `-f`         | `--package-root="."`                 |  Directory to search for YAML files{{< /table >}}

crossplane xpkg build` 命令会递归查找由 `--package-root` 设置的目录，并尝试将任何以 `.yml` 或 `.yaml` 结尾的文件合并为一个软件包。

所有 YAML 文件都必须是有效的 Kubernetes 配置清单，并包含 `apiVersion`、`kind`、`metadata` 和 `spec` 字段。

#### 忽略文件

使用 `--ignore` 可以提供要忽略的文件和目录列表。

例如，"crossplane xpkg build --ignore="./test/*,kind-config.yaml"`

#### 设置 package 名称

`crossplane` 会自动以 `metadata.name` 和软件包内容哈希值的组合为新软件包命名，并将内容保存在与 `--package-root` 相同的位置。 使用 `--package-file` 或 `-o` 定义特定位置和文件名。

例如，"crossplane xpkg build -o /home/crossplane/example.xpkg`"。

#### 包括实例

包含 YAML 文件，演示如何通过 `--examples-root` 使用 packages。

[upbound市场](https://marketplace.upbound.io/) 使用被引用为"--examples-root "的文件作为已发布软件包的文档。

#### 包括运行时镜像

函数和 Providers 需要描述其依赖关系和设置的 YAML 文件，以及运行时的容器镜像。

使用 `--embed-runtime-image-name`可运行指定镜像，并将镜像包含在函数或 Providers package 内。

{{<hint "note" >}}使用 `--embed-runtime-image-name` 引用的镜像必须在本地 Docker 缓存中。

使用 `docker pull` 下载丢失的镜像。{{< /hint >}}

embed-runtime-image-tarball "标志将本地 OCI 镜像压缩包包含在函数或 Providers 包内。

### xpkg install

使用 `crossplane xpkg install` 将软件包下载并安装到 crossplane 中。

默认情况下，"crossplane xpkg install "命令被引用在"~/.kube/config "中定义的 Kubernetes 配置。

使用环境变量 "KUBECONFIG "定义自定义 Kubernetes 配置文件位置。

指定软件包类型、软件包文件，还可选择在 crossplane 中为软件包命名。

`crossplane xpkg install<package-kind> <registry URL package name and tag> [<optional-name>]`

<package-kind> 可以是配置、功能或 Provider。

例如，要安装 0.42.0 版的 [AWS S3 Provider](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v0.42.0): 

`crossplane xpkg install provider xpkg.upbound.io/upbound/provider-aws-s3:v0.42.0`

#### flag

{{< table "table table-sm table-striped">}}<number of seconds>| Short flag   | Long flag                                        | Description                                                                                     |
| ------------ | -------------                                    | ------------------------------                                                                  |
|              | `--runtime-config=<runtime config name>`         | Install the package with a runtime configuration.                                               |
| `-m`         | `--manual-activation`                            | Set the `revisionActiviationPolicy` to `Manual`.                                                |
|              | `--package-pull-secrets=<list of secrets>`       | A comma-separated list of Kubernetes secrets to use for authenticating to the package registry. |
| `-r`         | `--revision-history-limit=<number of revisions>` | Set the `revisionHistoryLimit`. Defaults to `1`.                                                |
| `

{{< /table >}}

#### 等待软件包安装

安装软件包时，"crossplane xpkg install "命令不会等待软件包下载和安装。 使用 "kubectl describe configuration "检查 "配置"，查看是否存在下载或安装问题。

使用 `--wait`，可以让 `crossplane xpkg install` 命令等待软件包达到 `HEALTHY` 条件后再继续。 如果在软件包达到 `HEALTHY` 条件之前 `wait` 时间已过，命令将返回错误。

#### 需要手动激活 package

将软件包设置为 require [manual activation]({{<ref "../concepts/packages#revision-activation-policy" >}})，防止自动升级带有 `--manual-activation` 的软件包

#### 对私人注册表进行身份验证

要对私有软件包注册表进行身份验证，请引用 `--package-pull-secrets`，并提供 Kubernetes Secret 对象列表。

{{<hint "important" >}}secrets 必须与 crossplane pod 位于同一个 namespace 中。{{< /hint >}}

#### 自定义存储软件包版本的数量

默认情况下，crossplane 只在本地软件包缓存中存储单个非活动软件包。

使用 `--revision-history-limit` 存储更多软件包的非活动副本。

有关 [packages revisions]({{< ref "../concepts/packages#configuration-revisions" >}}) 在 packages 文档中。

### xpkg 登录

使用 `xpkg login` 对 [upbound Marketplace](https://marketplace.upbound.io/) 容器注册表 `xpkg.upbound.io` 进行身份验证。

[在 upbound Marketplace 注册](https://accounts.upbound.io/register) 来推送软件包和创建私有软件仓库。

#### flag

{{< table "table table-sm table-striped">}}| 短标志 | 长标志 | 说明 | | ------------ | ------------- | ------------------------------ | | `-u` | `-用户名=<username>| | 验证时要使用的用户名。 | | `-p` | `-密码=<password>| | 验证时要使用的密码。 | | `-t` | `-令牌=<token string>| | 验证时要使用的用户令牌字符串。 | | `-a` | `-账户=<organization>| | 验证时指定一个 upbound 组织。{{< /table >}}

#### 验证选项

crossplane xpkg login` 命令可以被引用用户名和密码或 upbound API 令牌。

默认情况下，"crossplane xpkg login "不带参数，会提示输入用户名和密码。

使用`--username`和`--password`标志提供用户名和密码，或设置环境变量`UP_USER`以获得用户名，或设置`UP_PASSWORD`以获得密码。

通过 `--token` 或 `UP_TOKEN` 环境变量，使用 upbound 用户令牌代替用户名和密码。

{{< hint "important" >}}-token "或 "UP_TOKEN "环境变量优先于用户名和密码。{{< /hint >}}

使用 `-` 作为 `--password` 或 `--token` 的输入，会从 stdin 读取输入。 例如，"crossplane xpkg login --password -`"。

登录后，crossplane CLI 会在 `.crossplane/config.json` 中创建一个 `profile` 来缓存非特权账户信息。

{{<hint "note" >}}config.json` 文件中的 `session` 字段是会话 cookie 标识符。

session "值不被引用用于身份验证。 这不是一个 "令牌"。{{< /hint >}}

#### 与注册的 upbound 组织进行身份验证

使用"--账户 "选项，连同用户名和密码或令牌，向 upbound Marketplace 中的注册组织进行身份验证。

例如，`crossplane xpkg login --account=Upbound --username=my-user --password -`。

### xpkg 注销

使用 `crossplane xpkg logout` 使当前的 `crossplane xpkg login` 会话失效。

{{< hint "note" >}}被引用 `crossplane xpkg logout` 会删除 `~/.crossplane/config.json` 文件中的 `session` ，但不会删除配置文件。{{< /hint >}}

### xpkg push

将 crossplane 软件包文件推送到软件包注册表。

Crossplane CLI 默认会将镜像推送到位于 `xpkg.upbound.io` 的 [Upbound Marketplace](https://marketplace.upbound.io/)。

{{< hint "note" >}}推送软件包可能需要使用 [`crossplane xpkg login`](#xpkg-login) 进行身份验证{{< /hint >}}

用 `crossplane xpkg push<package>` 指定组织、软件包名称和标签

默认情况下，该命令会在当前目录下查找要推送的单个 `.xpkg` 文件。

要推送多个文件或指定特定的 `.xpkg` 文件，请使用 `-f` flag。

例如，要将名为 "my-package "的本地软件包推送到 "crossplane-docs/my-package:v0.14.0"，请使用

crossplane xpkg push -f my-package.xpkg crossplane-docs/my-package:v0.14.0

要推送到其他软件包注册表，如 [DockerHub](https://hub.docker.com/)，请提供完整的 URL 和软件包名称。

例如，要将名为 "my-package "的本地包推送到 DockerHub 组织 "crossplane-docs/my-package:v0.14.0"，请使用: "crossplane xpkg push -f my-package.xpkg index.docker.io/crossplane-docs/my-package:v0.14.0`.

#### flag

{{< table "table table-sm table-striped">}}| 短 flag | 长 flag | 说明 | | ------------ | ------------- | ------------------------------ | | `-f` | `--package-files=PATH` | 用逗号分隔的要推送的 xpkg 文件列表。{{< /table >}}

### xpkg更新

crossplane xpkg update "命令下载并更新现有软件包。

默认情况下，"crossplane xpkg update "命令被引用"~/.kube/config "中定义的 Kubernetes 配置。

使用环境变量 "KUBECONFIG "定义自定义 Kubernetes 配置文件位置。

指定软件包种类、软件包文件，还可指定已安装在 crossplane 中的软件包名称。

`crossplane xpkg update<package-kind> <registry package name and tag> [<optional-name>]`

软件包文件必须是 [Upbound Marketplace](https://marketplace.upbound.io/) 上 `xpkg.upbound.io` 注册表中的组织、镜像和标签。

例如，要更新到版本 0.42.0 的 [AWS S3 Provider](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v0.42.0): 

crossplane xpkg 更新 Provider xpkg.upbound.io/upbound/provider-aws-s3:v0.42.0

## beta

crossplane `beta`命令是试验性的。 这些命令可能会在未来的发布版本中改变 flag、选项或输出。

crossplane 维护者可能会在未来发布的版本中推广或删除 `beta` 下的相应命令。

#### 测试版渲染

crossplane beta render "命令预览[合成资源]({{<ref "../concepts/composite-resources">}}) 应用任何 [合成函数]({{<ref "../concepts/composition-functions">}}).

{{< hint "important" >}}crossplane beta render "命令不应用[补丁和变换构成补丁]({{<ref "../concepts/patch-and-transform">}}).

该命令仅支持 "修补和变换 "功能。{{< /hint >}}

`crossplane beta render` 命令会连接到本地运行的 Docker 引擎，拉取并运行 Composition 功能。

{{<hint "important">}}运行 `crossplane beta render` 需要 [Docker](https://www.docker.com/)。{{< /hint >}}

Provider 一个复合资源、Composition 和 Composition 函数 YAML 定义，并附带在本地渲染 Output 的命令。

例如，"crossplane beta render XR.yaml Composition.yaml function.yaml

输出包括原始 Composition 资源和生成的 managed 资源。

{{<expand "An example render output" >}}

```yaml
---
apiVersion: nopexample.org/v1
kind: XBucket
metadata:
  name: test-xrender
status:
  bucketRegion: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: my-bucket
  generateName: test-xrender-
  labels:
    crossplane.io/composite: test-xrender
  ownerReferences:
  - apiVersion: nopexample.org/v1
    blockOwnerDeletion: true
    controller: true
    kind: XBucket
    name: test-xrender
    uid: ""
spec:
  forProvider:
    region: us-east-2
```

{{< /expand >}}

#### flag

{{< table "table table-sm table-striped">}}<directory or file>| 短标志 | 长标志 | 说明 | | ------------ | ------------- | ------------------------------ | | | `--context-files=<key>=<file>,<key>=<file>| 为函数 "上下文 "加载的以逗号分隔的文件列表。" | | | `--context-values=<key>=<value>,<key>=<value>| 为函数 "上下文 "加载的以逗号分隔的键值对列表。" | | `-r` | `--include-function-results` | 包括函数的 "结果 "或事件。{{< /table >}}

crossplane beta render` 命令依靠标准的 [Docker 环境变量](https://docs.docker.com/engine/reference/commandline/cli/#environment-variables) 连接到本地 Docker 引擎并运行 Composition 功能。

#### 提供功能上下文

`--context-files` 和 `--context-values` 标志可以为函数的 `context` 提供数据。 上下文是 JSON 格式的数据。

#### 包括函数结果

如果函数会产生带有状态的 Kubernetes 事件，请使用 `--include-function-results`，将它们与托管资源输出一起打印出来。

#### 模拟受管资源

用 `--observed-resources`提供代表托管资源的模拟或人工数据。 crossplane beta render`命令会将所提供的输入视为 Crossplane 集群中的资源。

函数在运行过程中可以引用和操作包含的资源。

观察到的资源 "可以是包含多个资源的单个 YAML 文件，也可以是代表多个资源的 YAML 文件目录。

在 YAML 文件中包含一个{{<hover label="apiVersion" line="1">}}apiVersion{{</hover>}},{{<hover label="apiVersion" line="2">}}类型{{</hover>}},{{<hover label="apiVersion" line="3">}}元数据{{</hover>}}和{{<hover label="apiVersion" line="7">}}规格{{</hover>}}.

```yaml {label="or"}
apiVersion: example.org/v1alpha1
kind: ComposedResource
metadata:
  name: test-render-b
  annotations:
    crossplane.io/composition-resource-name: resource-b
spec:
  coolerField: "I'm cooler!"
```

资源模式未经验证，可能包含任何数据。

###β 痕量

使用 `crossplane beta trace` 命令可显示 crossplane 对象的可视化关系。 `trace` 命令支持 claims、composition 或 managed resources。

该命令需要一个资源类型和一个资源名称。

` crossplane beta trace<resource kind> <resource name>`

例如，要查看类型为 `example.crossplane.io` 的名为 `my-claim` 的资源:  `crossplane beta trace example.crossplane.io my-claim`

该命令还接受 Kubernetes CLI 风格的 `<kind>/<name>` 输入。例如，`crossplane beta trace example.crossplane.io/my-claim`

默认情况下，"crossplane beta trace "命令被引用在"~/.kube/config "中定义的 Kubernetes 配置。

使用环境变量 "KUBECONFIG "定义自定义 Kubernetes 配置文件位置。

#### flag

{{< table "table table-sm table-striped">}}

<!-- vale Crossplane.Spelling = NO -->

<!-- vale flags `dot` as an error but only the trailing tick. -->

| 短标志 | 长标志 | 说明 | | ------------ | ------------- | ------------------------------ | | `-n` | `--namespace` | 资源的命名空间。 | | `-o` | `--output=` | 使用 `wide`、`json` 或 `dot` 更改图形输出，以获得 [Graphviz dot](https://graphviz.org/docs/layouts/dot/) 输出。 | | `-s` | `--show-connection-secrets` | 打印任何连接秘密名称。 不打印秘密值。

<!-- vale Crossplane.Spelling = YES -->

{{< /table >}}

#### 输出选项

默认情况下，"crossplane beta trace "直接打印到终端，将 "就绪 "条件和 "状态 "信息限制为 64 个字符。

以下示例输出了 AWS 参考平台的 "集群 "索赔，其中包括多个 Composition 和组成资源: 

```shell {copy-lines="1"}
crossplane beta trace cluster.aws.platformref.upbound.io platform-ref-aws
NAME SYNCED READY STATUS
Cluster/platform-ref-aws (default)                                True True Available
└─ XCluster/platform-ref-aws-mlnwb True True Available
   ├─ XNetwork/platform-ref-aws-mlnwb-6nvkx True True Available
   │  ├─ VPC/platform-ref-aws-mlnwb-ckblr True True Available
   │  ├─ InternetGateway/platform-ref-aws-mlnwb-r7w47 True True Available
   │  ├─ Subnet/platform-ref-aws-mlnwb-lhr4h True True Available
   │  ├─ Subnet/platform-ref-aws-mlnwb-bss4b True True Available
   │  ├─ Subnet/platform-ref-aws-mlnwb-fzbxx True True Available
   │  ├─ Subnet/platform-ref-aws-mlnwb-vxbf4 True True Available
   │  ├─ RouteTable/platform-ref-aws-mlnwb-cs9nl True True Available
   │  ├─ Route/platform-ref-aws-mlnwb-vpxdg True True Available
   │  ├─ MainRouteTableAssociation/platform-ref-aws-mlnwb-sngx5 True True Available
   │  ├─ RouteTableAssociation/platform-ref-aws-mlnwb-hprsp True True Available
   │  ├─ RouteTableAssociation/platform-ref-aws-mlnwb-shb8f True True Available
   │  ├─ RouteTableAssociation/platform-ref-aws-mlnwb-hvb2h True True Available
   │  ├─ RouteTableAssociation/platform-ref-aws-mlnwb-m58vl True True Available
   │  ├─ SecurityGroup/platform-ref-aws-mlnwb-xxbl2 True True Available
   │  ├─ SecurityGroupRule/platform-ref-aws-mlnwb-7qt56 True True Available
   │  └─ SecurityGroupRule/platform-ref-aws-mlnwb-szgxp True True Available
   ├─ XEKS/platform-ref-aws-mlnwb-fqjzz True True Available
   │  ├─ Role/platform-ref-aws-mlnwb-gmpqv True True Available
   │  ├─ RolePolicyAttachment/platform-ref-aws-mlnwb-t6rct True True Available
   │  ├─ Cluster/platform-ref-aws-mlnwb-crrt8 True True Available
   │  ├─ ClusterAuth/platform-ref-aws-mlnwb-dgn6f True True Available
   │  ├─ Role/platform-ref-aws-mlnwb-tdnx4 True True Available
   │  ├─ RolePolicyAttachment/platform-ref-aws-mlnwb-qzljh True True Available
   │  ├─ RolePolicyAttachment/platform-ref-aws-mlnwb-l64q2 True True Available
   │  ├─ RolePolicyAttachment/platform-ref-aws-mlnwb-xn2px True True Available
   │  ├─ NodeGroup/platform-ref-aws-mlnwb-4sfss True True Available
   │  ├─ OpenIDConnectProvider/platform-ref-aws-mlnwb-h26xx True True Available
   │  └─ ProviderConfig/platform-ref-aws                          -        -
   └─ XServices/platform-ref-aws-mlnwb-bgndx True True Available
      ├─ Release/platform-ref-aws-mlnwb-bcj7r True True Available
      └─ Release/platform-ref-aws-mlnwb-7hfkv True True Available
```

#### 宽幅输出

如果 "就绪 "或 "状态 "信息长度超过 64 个字符，则使用 `--output=wide`打印整个信息。

例如，Output 会截断过长的 "状态 "信息。

```shell {copy-lines="1"
crossplane trace cluster.aws.platformref.upbound.io platform-ref-aws
NAME SYNCED READY STATUS
Cluster/platform-ref-aws (default)                                True False Waiting: ...resource claim is waiting for composite resource to become Ready
```

被引用 `--output=wide` 可查看完整信息。

```shell {copy-lines="1"
crossplane trace cluster.aws.platformref.upbound.io platform-ref-aws --output=wide
NAME SYNCED READY STATUS
Cluster/platform-ref-aws (default)                                True False Waiting: Composite resource claim is waiting for composite resource to become Ready
```

#### Graphviz dot 文件输出

使用 `--output=dot` 可以打印出文本 [Graphviz dot](https://graphviz.org/docs/layouts/dot/)输出。

保存输出并将其导出，或将输出直接导入 Graphviz `dot` 以渲染镜像。

例如，要将输出保存为 `graph.png` 文件，请使用 `dot -Tpng -o graph.png`。

`crossplane beta trace集群.aws.platformref.upbound.io platform-ref-aws -o dot | dot -Tpng -o graph.png`

#### 打印连接秘密

使用 `-s` 可将任何连接 secret 名称与其他资源一起打印出来。

{{<hint "important">}}crossplane beta trace` 命令不打印 secret 值。{{< /hint >}}

输出结果包括秘密名称和秘密的 namespace。

```shell
NAME SYNCED READY STATUS
Cluster/platform-ref-aws (default)                                          True True Available
└─ XCluster/platform-ref-aws-mlnwb True True Available
   ├─ XNetwork/platform-ref-aws-mlnwb-6nvkx True True Available
   │  ├─ SecurityGroupRule/platform-ref-aws-mlnwb-szgxp True True Available
   │  └─ Secret/3f11c30b-dd94-4f5b-aff7-10fe4318ab1f (upbound-system)       -        -
   ├─ XEKS/platform-ref-aws-mlnwb-fqjzz True True Available
   │  ├─ OpenIDConnectProvider/platform-ref-aws-mlnwb-h26xx True True Available
   │  └─ Secret/9666eccd-929c-4452-8658-c8c881aee137-eks (upbound-system)   -        -
   ├─ XServices/platform-ref-aws-mlnwb-bgndx True True Available
   │  ├─ Release/platform-ref-aws-mlnwb-7hfkv True True Available
   │  └─ Secret/d0955929-892d-40c3-b0e0-a8cabda55895 (upbound-system)       -        -
   └─ Secret/9666eccd-929c-4452-8658-c8c881aee137 (upbound-system)          -        -
```

### beta xpkg init

`crossplane beta xpkg init` 命令会在当前目录下填充文件，以构建软件包。

Provider 被引用的软件包名称和软件包模板应从命令 `crossplane beta xpkg init<name> <template>`开始。

<name>` 输入没有被引用。Crossplane 为将来的发布保留了 `<name>`。

<template>` 值可以是四种众所周知的模板之一: 

* `function-template-go` - 一个用于构建 crossplane Go [组成函数]({{<ref "../concepts/composition-functions">}}) 的模板来自 [crossplane/function-template-go](https://github.com/crossplane/function-template-go) 资源库。
* `function-template-python` - 一个模板，用于从 [crossplane/function-template-go]() 资源库中创建 Crossplane Python [组成函数]({{<ref "../concepts/composition-functions">}}) 的模板，来自 [crossplane/function-template-python](https://github.com/crossplane/function-template-go) 资源库。
* `provider-template` - 从 [crossplane/provider-template](https://github.com/crossplane/provider-template) 资源库中创建基本的 Crossplane 提供程序的模板。
* `provider-template-upjet` - 用于从现有 Terraform 提供程序构建基于 [Upjet](https://github.com/crossplane/upjet) 的 Crossplane 提供程序的模板。复制自 [upbound/upjet-provider-template](https://github.com/upbound/upjet-provider-template) 资源库。

<template>` 值可以是 git 仓库的 URL，而不是众所周知的模板。

#### NOTES.txt

如果模板库的根目录中包含`NOTES.txt`文件，则`crossplane beta xpkg init`命令会在用模板文件填充目录后将该文件的内容打印到终端。 这对于提供有关模板的信息非常有用。

#### init.sh

如果模板库的根目录中包含`init.sh`文件，则`crossplane beta xpkg init`命令在向目录中填充模板文件后会启动一个对话框。 对话框会提示用户是否要查看或运行脚本。 使用初始化脚本可自动个性化模板。

#### flag

{{< table "table table-sm table-striped">}}| 短标志 | 长标志 | 说明 | | ------------ | ----------------------- | ------------------------------ | | `-d` | `--directory` | 创建和加载模板文件的目录。 默认被引用到当前目录。 | | `-r` | `--run-init-script` | 运行 init.sh 脚本，如果存在的话，无需提示。

<!-- vale Crossplane.Spelling = YES -->

{{< /table >}}
