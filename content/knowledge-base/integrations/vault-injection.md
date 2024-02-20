---

title: Vault 凭证注入
weight: 230

---

&gt; 本指南改编自 [Vault on Minikube](https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube) 和 [Vault Kubernetes &gt; Sidecar](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar) 指南。

大多数 crossplane 提供商至少支持提供以下来源的证书: 

* Kubernetes secret
* 环境变量
* 文件系统

提供程序可以选择性地支持其他凭证源，但常用的凭证源涵盖了各种各样的用例。 在使用 [Vault](https://www.vaultproject.io/) 进行secret管理的组织中，有一种特定用例很受欢迎，那就是使用侧卡将凭证注入文件系统。本指南将演示如何使用 [Vault Kubernetes Sidecar](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar) 为 [provider-gcp](https://marketplace.upbound.io/providers/crossplane-contrib/provider-gcp) 和 [provider-aws](https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws) 提供凭证。

&gt; 注: 在本指南中，我们将把 GCP 凭据和 AWS 访问密钥 &gt; 复制到 Vault 的 KV secret引擎中。 这是一种使用 Vault &gt; 管理secret的简单通用方法，但不如使用 Vault 的 &gt; [AWS](https://www.vaultproject.io/docs/secrets/aws)、[Azure](https://www.vaultproject.io/docs/secrets/azure) 和 [GCP](https://www.vaultproject.io/docs/secrets/gcp) 专用云提供商secret引擎那么强大。

## 设置

&gt; 注意: 本指南将介绍如何设置 Vault 与 &gt; Crossplane 在同一集群中运行。 您也可以选择使用现有的 Vault 实例，该实例在集群外运行，但已启用 Kubernetes 验证。

在开始之前，您必须确保已经安装了 crossplane 和 Vault，并且它们正在集群中运行。

1.安装 crossplane

```console
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

2.安装 Vault Helm 图表

```console
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault
```

3.解封 Vault 实例

为了让 Vault 从物理存储中访问加密数据，必须对其进行 [解封](https://www.vaultproject.io/docs/concepts/seal)。

```console
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

4.启用 Kubernetes 验证方法

为了让Vault能够根据Kubernetes服务账户验证请求，必须启用[Kubernetes身份验证后端](https://www.vaultproject.io/docs/auth/kubernetes)。这需要登录Vault并配置服务账户令牌、API服务器地址和证书。由于我们是在Kubernetes中运行Vault，这些值已经可以通过容器文件系统和环境变量获得。

```console
cat cluster-keys.json | jq -r ".root_token" # get root token

kubectl exec -it vault-0 -- /bin/sh
vault login # use root token from above
vault auth enable kubernetes

vault write auth/kubernetes/config \
        token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
        kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

5.出口 Vault 集装箱

接下来的步骤将在本地环境中执行。

```console
exit
```

{{< tabs >}}
{{< tab "GCP" >}}

## 创建 GCP 服务账户

为了在 GCP 上配置基础设施，您需要创建一个具有适当权限的服务账户。 在本指南中，我们将只配置一个 CloudSQL 实例，因此服务账户将绑定到 "cloudsql.admin "角色。 以下步骤将设置一个 GCP 服务账户，赋予它必要的权限，以便 crossplane 能够管理 CloudSQL 实例，并在 JSON 文件中发出服务账户凭据。

```console
# replace this with your own gcp project id and the name of the service account
# that will be created.
PROJECT_ID=my-project
NEW_SA_NAME=test-service-account-name

# create service account
SA="${NEW_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create $NEW_SA_NAME --project $PROJECT_ID

# enable cloud API
SERVICE="sqladmin.googleapis.com"
gcloud services enable $SERVICE --project $PROJECT_ID

# grant access to cloud API
ROLE="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding --role="$ROLE" $PROJECT_ID --member "serviceAccount:$SA"

# create service account keyfile
gcloud iam service-accounts keys create creds.json --project $PROJECT_ID --iam-account $SA
```

现在，您应该在 `creds.json` 中获得有效的服务帐户凭据。

## 将凭据存储在 Vault 中

设置 Vault 后，您需要在 [kv secrets engine](https://www.vaultproject.io/docs/secrets/kv/kv-v2) 中存储凭证。

&gt; 注意: 以下步骤涉及将凭证复制到容器 &gt; 文件系统，然后再存储到 Vault 中。 您也可以选择将容器端口转发到本地环境 &gt;（`kubectl port-forward vault-0 8200:8200`），从而使用 Vault 的 &gt; HTTP API 或用户界面。

1.将凭证文件复制到 Vault 容器中

将凭据复制到容器文件系统中，以便存储在 Vault 中。

```console
kubectl cp creds.json vault-0:/tmp/creds.json
```

2.启用 KV secret引擎

secret引擎必须启用后才能被引用。 在 `secret` 路径下启用 `kv-v2` secret引擎。

```console
kubectl exec -it vault-0 -- /bin/sh

vault secrets enable -path=secret kv-v2
```

3.在 KV 引擎中存储 GCP 凭据

将secret注入到 `provider-gcp` 控制器 `Pod` 时，您的 GCP 凭据的路径将被引用。

```console
vault kv put secret/provider-creds/gcp-default @tmp/creds.json
```

4.清理证书文件

您不再需要容器文件系统中的 GCP 凭据文件，所以请继续清理它。

```console
rm tmp/creds.json
```

{{< /tab >}}
{{< tab "AWS" >}}

## 创建 AWS IAM 用户

为了在 AWS 上调配基础设施，您需要使用现有的或创建具有适当权限的新 IAM 用户。 以下步骤将创建一个 AWS IAM 用户，并赋予其必要的权限。

&gt; 注意: 如果您已有一个具有适当权限的 IAM 用户，则可以跳过此步骤，但仍需 &gt; Provider `ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY` 环境变量的值。

```console
# create a new IAM user
IAM_USER=test-user
aws iam create-user --user-name $IAM_USER

# grant the IAM user the necessary permissions
aws iam attach-user-policy --user-name $IAM_USER --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# create a new IAM access key for the user
aws iam create-access-key --user-name $IAM_USER > creds.json
# assign the access key values to environment variables
ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId creds.json)
AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey creds.json)
```

## 将凭据存储在 Vault 中

设置 Vault 后，您需要在 [kv secrets engine](https://www.vaultproject.io/docs/secrets/kv/kv-v2) 中存储凭证。

1.启用 KV secret引擎

secret引擎必须启用后才能被引用。 在 `secret` 路径下启用 `kv-v2` secret引擎。

```console
kubectl exec -it vault-0 -- env \
  ACCESS_KEY_ID=${ACCESS_KEY_ID} \
  AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
  /bin/sh

vault secrets enable -path=secret kv-v2
```

2.在 KV 引擎中存储 AWS 凭据

在将secret注入到 `provider-aws` 控制器 `Pod` 时，您的 AWS 凭据的路径将被引用。

```
vault kv put secret/provider-creds/aws-default access_key="$ACCESS_KEY_ID" secret_key="$AWS_SECRET_ACCESS_KEY"
```

{{< /tab >}}
{{< /tabs >}}

## 为读取 Provider 凭据创建 Vault 策略

为了让我们的控制器能让 Vault sidecar 将凭证注入其文件系统，您必须将 `Pod` 与 [policy](https://www.vaultproject.io/docs/concepts/policies) 关联。该策略将允许在 `kv-v2` secrets 引擎中读取和列出 `provider-creds` 路径上的所有 secrets。

```console
vault policy write provider-creds - <<EOF
path "secret/data/provider-creds/*" {
    capabilities = ["read", "list"]
}
EOF
```

## 为 Crossplane Provider Pods 创建角色

1.创建角色

最后一步是创建一个与你创建的策略绑定的角色，并将其与一组 Kubernetes 服务账户关联。 该角色可由 `crossplane-system` 名称空间中的任何 (`*`) 服务账户承担。

```console
vault write auth/kubernetes/role/crossplane-providers \
        bound_service_account_names="*" \
        bound_service_account_namespaces=crossplane-system \
        policies=provider-creds \
        ttl=24h
```

2.出口 Vault 集装箱

接下来的步骤将在本地环境中执行。

```console
exit
```

{{< tabs >}}
{{< tab "GCP" >}}

## 安装 Provider-gcp

现在您已准备好安装 `provider-gcp`。 Crossplane 提供了一种 `ControllerConfig` 类型，允许您自定义提供程序控制器 `Pod` 的部署。 `ControllerConfig` 可由任意数量的希望使用其配置的 `Provider` 对象创建和引用。 在下面的示例中，`Pod` 注释向 Vault mutating webhook 表示，我们希望将存储在 `secret/provider-creds/gcp-default` 的secret以 `crossplane-providers` 角色注入容器文件系统。 此外，还添加了模板格式，以确保secret数据以 `provider-gcp` 期望的形式呈现。

{% raw %}

```console
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: vault-config
spec:
  metadata:
    annotations:
      vault.hashicorp.com/agent-inject: \"true\"
      vault.hashicorp.com/role: "crossplane-providers"
      vault.hashicorp.com/agent-inject-secret-creds.txt: "secret/provider-creds/gcp-default"
      vault.hashicorp.com/agent-inject-template-creds.txt: |
        {{- with secret \"secret/provider-creds/gcp-default\" -}}
         {{ .Data.data | toJSON }}
        {{- end -}}
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.22.0
  controllerConfigRef:
    name: vault-config" | kubectl apply -f -
```

{% endraw %}

## 配置 Provider-gcp

安装并运行 `provider-gcp` 后，您需要创建一个 `ProviderConfig` 来指定文件系统中的凭据，这些凭据应被用于调配引用此 `ProviderConfig` 的托管资源。 由于此 `ProviderConfig` 的名称是 `default` ，它将被任何未明确引用 `ProviderConfig` 的托管资源所使用。

&gt; 注意: 请确保之前定义的 `PROJECT_ID` 环境变量 &gt; 设置正确。

```console
echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: ${PROJECT_ID}
  credentials:
    source: Filesystem
    fs:
      path: /vault/secrets/creds.txt" | kubectl apply -f -
```

要验证 GCP 凭据是否已注入容器，请运行以下命令: 

```console
PROVIDER_CONTROLLER_POD=$(kubectl -n crossplane-system get pod -l pkg.crossplane.io/provider=provider-gcp -o name --no-headers=true)
kubectl -n crossplane-system exec -it $PROVIDER_CONTROLLER_POD -c provider-gcp -- cat /vault/secrets/creds.txt
```

## 提供基础设施

最后一步是实际配置一个 "CloudSQLInstance"。 创建以下对象将在 GCP 上创建一个云 SQL Postgres 数据库。

```console
echo "apiVersion: database.gcp.crossplane.io/v1beta1
kind: CloudSQLInstance
metadata:
  name: postgres-vault-demo
spec:
  forProvider:
    databaseVersion: POSTGRES_12
    region: us-central1
    settings:
      tier: db-custom-1-3840
      dataDiskType: PD_SSD
      dataDiskSizeGb: 10
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: cloudsqlpostgresql-conn" | kubectl apply -f -
```

您可以使用以下命令监控数据库配置的进度: 

```console
kubectl get cloudsqlinstance -w
```

{{< /tab >}}
{{< tab "AWS" >}}

## 安装 Provider-aws

现在您已准备好安装 `provider-aws`。 Crossplane 提供了一种 `ControllerConfig` 类型，允许您自定义提供程序控制器 `Pod` 的部署。 `ControllerConfig` 可由任意数量希望使用其配置的 `Provider` 对象创建和引用。 在下面的示例中， `Pod` 注解向 Vault mutating webhook 表示，我们希望将存储在 `secret/provider-creds/aws-default` 中的secret通过担任 `crossplane-providers` 角色注入容器文件系统。 还添加了一些模板格式，以确保secret数据以 `provider-aws` 期望的形式呈现。

{% raw %}

```console
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: aws-vault-config
spec:
  args:
    - --debug
  metadata:
    annotations:
      vault.hashicorp.com/agent-inject: \"true\"
      vault.hashicorp.com/role: \"crossplane-providers\"
      vault.hashicorp.com/agent-inject-secret-creds.txt: \"secret/provider-creds/aws-default\"
      vault.hashicorp.com/agent-inject-template-creds.txt: |
        {{- with secret \"secret/provider-creds/aws-default\" -}}
          [default]
          aws_access_key_id="{{ .Data.data.access_key }}"
          aws_secret_access_key="{{ .Data.data.secret_key }}"
        {{- end -}}
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: aws-vault-config" | kubectl apply -f -
```

{% endraw %}

## 配置 Provider-aws

安装并运行 "provider-aws "后，您需要创建一个 "ProviderConfig"，指定文件系统中的凭据，用于调配引用该 "ProviderConfig "的托管资源。 由于该 "ProviderConfig "的名称是 "default"，因此将被任何未明确引用 "ProviderConfig "的托管资源所使用。

```console
echo "apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Filesystem
    fs:
      path: /vault/secrets/creds.txt" | kubectl apply -f -
```

要验证 AWS 凭据是否已注入容器，请运行以下命令: 

```console
PROVIDER_CONTROLLER_POD=$(kubectl -n crossplane-system get pod -l pkg.crossplane.io/provider=provider-aws -o name --no-headers=true)
kubectl -n crossplane-system exec -it $PROVIDER_CONTROLLER_POD -c provider-aws -- cat /vault/secrets/creds.txt
```

## 提供基础设施

最后一步是实际配置一个 "桶"。 创建以下对象将在 AWS 上创建一个 S3 桶。

```console
echo "apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: s3-vault-demo
spec:
  forProvider:
    acl: private
    locationConstraint: us-east-1
    publicAccessBlockConfiguration:
      blockPublicPolicy: true
    tagging:
      tagSet:
        - key: Name
          value: s3-vault-demo
  providerConfigRef:
    name: default" | kubectl apply -f -
```

您可以使用以下命令监控水桶调配的进度: 

```console
kubectl get bucket -w
```

{{< /tab >}}
{{< /tabs >}}

<!-- named links -->