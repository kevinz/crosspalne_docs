---

weight: 400
title: crossplane CLI
description: "Crossplane 命令行界面的文档"

---

Crossplane CLI 有助于简化 Crossplane 的某些开发和管理环节。

crossplane CLI 包括

* 构建、安装、更新和推送 crossplane 软件包的工具
* 独立进行 Composition 功能测试和渲染，无需访问运行 Crossplane 的 Kubernetes 集群
* 排除 Crossplane Composition、复合资源和托管资源的故障

## 安装 CLI

crossplane CLI 是一个独立的二进制文件，没有外部依赖性。

{{<hint "note" >}}在用户电脑上安装 crossplane CLI。

大多数 crossplane CLI 命令都独立于 Kubernetes，不需要访问 crossplane pod。{{< /hint >}}

使用 crossplane 安装脚本下载适用于您 CPU 架构的最新版本。

```shell
curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | sh
```

[脚本](https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh) 会检测你的 CPU 架构，并下载最新的稳定版本。

{{<expand "Manually install the Crossplane CLI" >}}

如果不想运行 shell 脚本，可以从 https://releases.crossplane.io/stable/current/bin 的 crossplane 发布库中手动下载二进制文件。

{{<hint "important" >}}

<!-- vale write-good.Passive = NO -->

CLI 在发布库中名为 `crank`。 下载此文件。

<!-- vale write-good.Passive = YES -->

crossplane "二进制文件是 Kubernetes Crossplane pod 镜像。{{< /hint >}}

将二进制文件移至 `$PATH` 中的某个位置，例如 `/usr/local/bin`。{{< /expand >}}

### 下载其他 CLI 版本

使用 `XP_CHANNEL` 和 `XP_VERSION` 环境变量下载不同的 crossplane CLI 版本或不同的发布分支。

默认情况下，CLI 从名为 `stable` 的 `XP_CHANNEL` 和 `XP_VERSION` 的 `current`安装，以匹配最新的稳定发布版本。

例如，要安装 CLI 版本`v1.14.0`，请在下载脚本 curl 命令中添加`XP_VERSION=v1.14.0`: 

`curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | XP_VERSION=v1.14.0 sh`