---

title: 自签名 CA 证书
weight: 270

---

&gt; 不建议在生产中使用自签名证书，它是

建议只使用自签名证书进行测试。

当 Crossplane 从私有注册表加载配置和 Provider 包时，必须将其配置为信任 CA 和中间认证。

crossplane 需要通过 helm chart 安装，并定义`registryCaBundleConfig.name`和`registryCaBundleConfig.key`参数。 参见 [Install Crossplane]({{<ref "../../master/software/install" >}}).

## 配置

1.创建 CA 捆绑包（一个包含根 CA 和中间 CA 的文件）。

这可以通过任何文本编辑器或命令行完成，只要生成的文件中包含的所有 crt 文件顺序正确即可。 在许多情况下，这要么是一个自签名的根 CA crt 文件，要么是一个中级 crt 文件和根 crt 文件。 crt 文件的顺序应按签名顺序从低到高排列。 例如，如果根证书下面有两个证书链，则应将最底层的中级证书放在文件开头，然后是签署该证书的中级证书，最后是签署该证书的根证书。

2.将文件保存为 `[yourdomain].ca-bundle`。
3.在你的 crossplane 系统命名空间中创建 Kubernetes ConfigMap: 

```
kubectl -n [Crossplane system namespace] create cm ca-bundle-config \
--from-file=ca-bundle=./[yourdomain].ca-bundle
```

4.将 `registryCaBundleConfig.name` helm chart 参数设置为

`ca-bundle-config` 和 `ca-bundle` 的 `registryCaBundleConfig.key` 参数。

&gt; Helm 文档中介绍了如何为 Helm 提供参数值、

[Helm install](https://helm.sh/docs/helm/helm_install/)。`override.yaml` 文件中的示例块如下: 

```
registryCaBundleConfig:
    name: ca-bundle-config
    key: ca-bundle
```