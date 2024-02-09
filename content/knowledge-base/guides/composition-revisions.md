---

title: 构成修订
状态:  beta
alphaVersion: "1.4"
betaVersion: "1.11"

---

本指南讨论如何使用 "Composition Revisions "安全地对 crossplane [`Composition`][组合类型]进行修改和回滚。 本指南假定读者熟悉 crossplane，特别是 [Composition]。

Composition "配置了 Crossplane 应该如何调和复合资源（XR）。 换句话说，当你创建一个 XR 时，所选的 "Composition "决定了 Crossplane 将创建哪些管理资源作为响应。 例如，假设你定义了一个 "PlatformDB "XR，它代表了你的组织对 Azure MySQL 服务器和一些防火墙规则的通用数据库配置。 Composition "包含了 MySQL 服务器和防火墙规则的 "基本 "配置，这些配置由 "PlatformDB "的配置扩展自。

Composition "和使用它的 XR 之间是一对多的关系。 您可能会定义一个名为 "big-platform-db "的 "Composition"，它被十个不同的 "PlatformDB "XR 使用。 通常，为了实现自助服务，"Composition "由不同于实际 "PlatformDB "XR 的团队管理。 例如，"Composition "可能由平台团队成员编写和维护，而各个应用程序团队则创建使用所述 "Composition "的 "PlatformDB "XR。

每个 "Composition "都是可变的--您可以根据组织需求的变化对其进行更新。 然而，如果没有 "Composition Revisions"，更新 "Composition "将是一个充满风险的过程。 Crossplane会不断使用 "Composition "来确保您的实际基础设施--您的MySQL服务器和防火墙规则--与您所期望的状态相匹配。 如果您有10个 "PlatformDB "XRs都引用了 "big-platform-db""Composition"，那么所有这10个XRs都将根据您对 "big-platform-db""Composition "所做的任何更新进行即时更新。

Composition Revisions 允许 XRs 选择退出自动更新。 相反，您可以按照自己的节奏更新 XRs 以利用最新的 "Composition "设置。 这样，您就可以对基础架构进行[金丝雀]更改，或将某些 XRs 回滚到以前的 "Composition "设置，而无需回滚所有 XRs。

### 被引用的作文修改

启用 "Composition Revisions "后，会发生三件事: 

1.Crossplane 会为每次 "Composition "更新创建一个 "CompositionRevision"。
2.Composition 资源会获得一个 `spec.compositionRevisionRef` 字段，用于指定它们使用的 `CompositionRevision` 版本。
3.复合资源会获得一个`spec.compositionUpdatePolicy`字段，用于指定应如何将其更新为新的 Composition Revisions。

每次编辑 "Composition "时，crossplane 都会自动创建一个 "CompositionRevision"，代表 "Composition "的 "修订版"--即唯一的状态。 每个修订版都会分配一个递增的修订版编号。 这样，"CompositionRevision "的用户就能知道哪个修订版是 "最新的"。

Crossplane 会区分 "Composition "的 "最新 "和 "当前 "修订版本，也就是说，如果你将 "Composition "还原到与现有 "CompositionRevision "相对应的先前状态，即使该修订版本不是 "最新 "修订版本（即最新的_唯一_"Composition "配置），它也会成为 "当前 "修订版本。

你可以使用 `kubectl` 发现哪些修订被引用: 

```console
# Find all revisions of the Composition named 'example'
kubectl get compositionrevision -l crossplane.io/composition-name=example
```

这样就会产生类似的 Output 输出: 

```console
NAME REVISION CURRENT AGE
example-18pdg 1 False 4m36s
example-2bgdr 2 True 73s
example-xjrdm 3 False 61s
```

&gt; 每个 `CompositionRevision` 都是特定时间点上这些 &gt; 需求的不可变快照。

无论是否启用 "Composition Revisions"，默认情况下 crossplane 的行为都是一样的。 这是因为启用 "Composition Revisions "后，所有 XR 都默认采用 "自动""CompositionUpdatePolicy"。 XR 支持两种更新策略: 

* 自动自动被引用当前的 `CompositionRevision`.(默认值）
* 手动需要人工干预才能更改 `CompositionRevision`.

下面的 XR 使用 "手动 "策略。 使用该策略时，XR 将在首次创建时选择当前的 "CompositionRevision"，但如果希望使用另一个 "CompositionRevision"，则必须手动更新。

```yaml
apiVersion: example.org/v1alpha1
kind: PlatformDB
metadata:
  name: example
spec:
  parameters:
    storageGB: 20
  # The Manual policy specifies that you do not want this XR to update to the
  # current CompositionRevision automatically.
  compositionUpdatePolicy: Manual
  compositionRef:
    name: example
  writeConnectionSecretToRef:
    name: db-conn
```

无论您选择了哪种 "CompositionUpdatePolicy"，XR 的 "compositionRevisionRef "都会在创建时自动设置。 如果您选择了 "Manual "策略，当您希望 XR 使用不同的 "CompositionRevision "时，必须编辑 "compositionRevisionRef "字段。

```yaml
apiVersion: example.org/v1alpha1
kind: PlatformDB
metadata:
  name: example
spec:
  parameters:
    storageGB: 20
  compositionUpdatePolicy: Manual
  compositionRef:
    name: example
  # Update the referenced CompositionRevision if and when you are ready.
  compositionRevisionRef:
    name: example-18pdg
  writeConnectionSecretToRef:
    name: db-conn
```

[作曲类型]:  {{<ref "../../master/concepts/compositions" >}}[Composition]:  {{<ref "../../master/concepts/compositions" >}}[金丝雀]:  https://martinfowler.com/bliki/CanaryRelease.html [install-guide]:  {{<ref "../../master/software/install" >}}