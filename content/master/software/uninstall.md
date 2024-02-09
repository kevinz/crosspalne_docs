---

title: 卸载 crossplane
weight: 300

---

{{<hint "warning" >}}如果没有按顺序卸载 Crossplane，Crossplane 创建的资源不会被删除。

这可能导致云资源运行，需要手动删除。{{< /hint >}}

## 下令卸载 crossplane

大多数 crossplane 资源都依赖于其他 crossplane 资源。

例如，_managed resource_ 依赖于_provider_。

如果不按顺序删除 crossplane 资源，可能会导致 crossplane 无法删除已供应的外部资源。

删除 crossplane 资源应按以下顺序进行: 

1.删除所有_复合资源定义_
2.删除所有剩余的_受管资源
3.移除所有_提供程序

删除 crossplane pod 会删除剩余的 crossplane 组件，如 _claims_。

{{<hint "tip" >}}使用 `kubectl get managed` 收集所有外部资源的清单。

根据 Kubernetes API 服务器的大小和资源数量，该命令可能需要几分钟才能返回。

{{<expand "An example kubectl get managed" >}}

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED EXTERNAL-NAME AGE
securitygroup.ec2.aws.upbound.io/my-db-7mc7h-j84h8 True True sg-0da6e9c29113596b6 3m1s
securitygroup.ec2.aws.upbound.io/my-db-8bhr2-9wsx9 True True sg-02695166f010ec05b 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
route.ec2.aws.upbound.io/my-db-7mc7h-vw985 True True r-rtb-05822b8df433e4e2b1080289494 3m1s
route.ec2.aws.upbound.io/my-db-8bhr2-7m2wq False True 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
securitygrouprule.ec2.aws.upbound.io/my-db-7mc7h-mkd9s True True sgrule-778063708 3m1s
securitygrouprule.ec2.aws.upbound.io/my-db-8bhr2-lzr89 False True 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
routetable.ec2.aws.upbound.io/my-db-7mc7h-mnqvm True True rtb-05822b8df433e4e2b 3m1s
routetable.ec2.aws.upbound.io/my-db-8bhr2-dfhj6 True True rtb-02e875abd25658254 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
subnet.ec2.aws.upbound.io/my-db-7mc7h-7m49d True True subnet-0c1ab32c5ec129dd1 3m2s
subnet.ec2.aws.upbound.io/my-db-7mc7h-9t64t True True subnet-07075c17c7a72f79e 3m2s
subnet.ec2.aws.upbound.io/my-db-7mc7h-rs8t8 True True subnet-08e88e826a42e55b4 3m2s
subnet.ec2.aws.upbound.io/my-db-8bhr2-9sjpx True True subnet-05d21c7b52f7ac8ca 2m26s
subnet.ec2.aws.upbound.io/my-db-8bhr2-dvrxf True True subnet-0432310376b5d09de 2m26s
subnet.ec2.aws.upbound.io/my-db-8bhr2-t7dpr True True subnet-0080fdcb6e9b70632 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
vpc.ec2.aws.upbound.io/my-db-7mc7h-ktbbh True True vpc-08d7dd84e0c12f33e 3m3s
vpc.ec2.aws.upbound.io/my-db-8bhr2-mrh2x True True vpc-06994bf323fc1daea 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
internetgateway.ec2.aws.upbound.io/my-db-7mc7h-s2x4v True True igw-0189c4da07a3142dc 3m1s
internetgateway.ec2.aws.upbound.io/my-db-8bhr2-q7dzl True True igw-01bf2a1dbbebf6a27 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-28qb4 True True rtbassoc-0718d680b5a0e68fe 3m1s
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-9hdlr True True rtbassoc-0faaedb88c6e1518c 3m1s
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-txhmz True True rtbassoc-0e5010724ca027864 3m1s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-bvgkt False True 2m26s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-d9gbg False True 2m26s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-k6k8m False True 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
instance.rds.aws.upbound.io/my-db-7mc7h-5d6w4 False True my-db-7mc7h-5d6w4 3m1s
instance.rds.aws.upbound.io/my-db-8bhr2-tx9kf False True my-db-8bhr2-tx9kf 2m26s

NAME READY SYNCED EXTERNAL-NAME AGE
subnetgroup.rds.aws.upbound.io/my-db-7mc7h-8c8n9 True True my-db-7mc7h-8c8n9 3m2s
subnetgroup.rds.aws.upbound.io/my-db-8bhr2-mc5ps True True my-db-8bhr2-mc5ps 2m27s

NAME READY SYNCED EXTERNAL-NAME AGE
bucket.s3.aws.upbound.io/crossplane-bucket-867737b10 True True
crossplane-bucket-867737b10 5m26s
```

{{</expand >}}
{{< /hint >}}

### 删除 Composition 资源定义

删除已安装的_复合资源定义_会删除_复合资源定义_所定义的任何_复合资源_及其创建的_受管资源_。

使用 `kubectl get xrd` 查看已安装的_复合资源定义_。

```shell {copy-lines="1"}
kubectl get xrd
NAME ESTABLISHED OFFERED AGE
compositepostgresqlinstances.database.example.org True True 40s
```

使用 `kubectl delete xrd` 删除_复合资源定义_。

```shell
kubectl delete xrd compositepostgresqlinstances.database.example.org
```

#### 删除托管资源

手动删除任何手动创建的_受管资源_。

使用 `kubectl get managed` 查看剩余的 _managed 资源。

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED EXTERNAL-NAME AGE
bucket.s3.aws.upbound.io/crossplane-bucket-867737b10 True True crossplane-bucket-867737b10 8h
```

被引用 `kubectl delete` 删除资源。

```shell
kubectl delete bucket.s3.aws.upbound.io/crossplane-bucket-867737b10
```

#### 移除 Crossplane Providers

使用 `kubectl get Providers` 列出已安装的_provider_。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
upbound-provider-aws True True xpkg.upbound.io/upbound/provider-aws:v0.27.0 8h
```

使用 `kubectl delete provider` 删除已安装的_provider_。

```shell
kubectl delete provider upbound-provider-aws
```

## 卸载 crossplane 部署

使用 `helm uninstall` 被引用卸载 crossplane

```shell
helm uninstall crossplane --namespace crossplane-system
```

使用 `kubectl get pods` 验证 Helm 是否删除了 crossplane pods

```shell
kubectl get pods -n crossplane-system
No resources found in crossplane-system namespace.
```

## 删除 crossplane 名称空间

Helm 安装 crossplane 时会创建 `crossplane-system` 命名空间。 Helm 不会使用 `helm uninstall` 卸载该命名空间。

使用 `kubectl delete namespace` 手动删除 crossplane 命名空间。

```shell
kubectl delete namespace crossplane-system
```

使用 `kubectl get namespaces` 验证 Kubernetes 是否删除了名称空间

```shell
kubectl get namespace
NAME STATUS AGE
default Active 2m45s
kube-flannel Active 2m42s
kube-node-lease Active 2m47s
kube-public Active 2m47s
kube-system Active 2m47s
```