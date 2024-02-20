---

title: 作为外部secret存储的 Vault
weight: 230

---

本指南介绍了配置 Crossplane 及其 Provider 将 [Vault](https://www.vaultproject.io/) 用作 [外部secret存储](https://github.com/crossplane/crossplane/blob/master/design/design-doc-external-secret-stores.md) (`ESS`)与 [ESS 插件 Vault](https://github.com/crossplane-contrib/ess-plugin-vault)所需的步骤。

{{<hint "warning" >}}外部secret存储是 alpha 功能。

crossplane 默认禁用外部secret存储。{{< /hint >}}

crossplane 会使用敏感信息，包括 Provider 凭据、对托管资源的输入和连接详情。

Vault 凭据注入指南]({{<ref "vault-injection" >}}) 详细介绍了如何使用 Vault 和 crossplane 注入 Provider 凭据。

Crossplane 不支持将 Vault 用于托管资源输入。[Crossplane issue #2985](https://github.com/crossplane/crossplane/issues/2985)跟踪对该功能的支持。

使用 Vault 支持连接详情需要一个 Crossplane 外部secret存储。

## 先决条件

本指南需要 [Helm](https://helm.sh) 3.11 或更高版本。

## 安装 Vault

{{<hint "note" >}}有关 [安装 Vault](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)的详细说明，可查阅 Vault 文档。{{< /hint >}}

### 添加 Vault Helm 图表

为 `hashicorp` 添加 Helm 资源库。

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com --force-update
```

使用 Helm 安装 Vault。

```shell
helm -n vault-system upgrade --install vault hashicorp/vault --create-namespace
```

### 打开 Vault 的封印

如果 Vault 已被 [密封](https://developer.hashicorp.com/vault/docs/concepts/seal)，则使用解封键解封 Vault。

获取 Vault 钥匙。

```shell
kubectl -n vault-system exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

用钥匙打开 Vault 的封条。

```shell {copy-lines="1"}
kubectl -n vault-system exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
Key Value
---             -----
Seal Type shamir
Initialized true
Sealed false
Total Shares 1
Threshold 1
Version 1.13.1
Build Date 2023-03-23T12:51:35Z
Storage Type file
Cluster Name vault-cluster-df884357
Cluster ID b3145d26-2c1a-a7f2-a364-81753033c0d9
HA Enabled false
```

## 配置 Vault Kubernetes 身份验证

为 Vault 启用[Kubernetes auth method](https://www.vaultproject.io/docs/auth/kubernetes)，以便根据 Kubernetes 服务账户验证请求。

### 获取 Vault 根令牌

Vault 根令牌位于[解封 Vault](#unseal-vault) 时创建的 JSON 文件内。

```shell
cat cluster-keys.json | jq -r ".root_token"
```

### 启用 Kubernetes 身份验证

连接到 Vault pod 中的外壳。

```shell {copy-lines="1"}
kubectl -n vault-system exec -it vault-0 -- /bin/sh
/ $
```

从 Vault 外壳，使用 _root 令牌登录 Vault。

```shell {copy-lines="1"}
vault login # use the root token from above
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key Value
---                  -----
token hvs.TSN4SssfMBM0HAtwGrxgARgn
token_accessor qodxHrINVlRXKyrGeeDkxnih
token_duration       ∞
token_renewable false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

在 Vault 中启用 Kubernetes 身份验证方法。

```shell {copy-lines="1"}
vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```

配置 Vault 与 Kubernetes 通信并退出 Vault shell

```shell {copy-lines="1-4"}
vault write auth/kubernetes/config \
        token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
        kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
/ $ exit
```

## 配置 Vault 以实现 Crossplane 集成

crossplane 依赖 Vault 键值secret引擎存储信息，Vault 需要为 crossplane 服务账户设置权限策略。

<!-- vale Crossplane.Spelling = NO -->

<!-- allow "kv" -->

### 启用 Vault kv secret 引擎

<!-- vale Crossplane.Spelling = YES -->

启用 [Vault KV Secrets Engine]（https://developer.hashicorp.com/vault/docs/secrets/kv）。

{{< hint "important" >}}Vault 有两个版本的 [KV Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/kv)。本例被引用的是版本 2。{{</hint >}}

```shell {copy-lines="1"}
kubectl -n vault-system exec -it vault-0 -- vault secrets enable -path=secret kv-v2
Success! Enabled the kv-v2 secrets engine at: secret/
```

### 为 crossplane 创建 Vault 策略

创建 Vault 策略，允许 crossplane 从 Vault 读写数据。

```shell {copy-lines="1-8"}
kubectl -n vault-system exec -i vault-0 -- vault policy write crossplane - <<EOF
path "secret/data/*" {
    capabilities = ["create", "read", "update", "delete"]
}
path "secret/metadata/*" {
    capabilities = ["create", "read", "update", "delete"]
}
EOF
Success! Uploaded policy: crossplane
```

将策略应用到 Vault。

```shell {copy-lines="1-5"}
kubectl -n vault-system exec -it vault-0 -- vault write auth/kubernetes/role/crossplane \
    bound_service_account_names="*" \
    bound_service_account_namespaces=crossplane-system \
    policies=crossplane \
    ttl=24h
Success! Data written to: auth/kubernetes/role/crossplane
```

## 安装 crossplane

{{<hint "important" >}}Crossplane v1.12 引入了插件支持，请确保您的 Crossplane 版本支持插件。{{< /hint >}}

在启用外部存储功能的情况下安装 crossplane。

```shell 
helm upgrade --install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace --set args='{--enable-external-secret-stores}'
```

## 安装 Crossplane Vault 插件

Crossplane Vault 插件不是 Crossplane 默认安装的一部分。 该插件以独特 Pod 的形式安装，使用 [Vault Agent Sidecar Injection](https://www.vaultproject.io/docs/platform/k8s/injector) 将 Vault secret存储连接到 Crossplane。

首先，为 Vault 插件 pod 配置 Annotations。

```yaml
cat > values.yaml <<EOF
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-token: "true"
  vault.hashicorp.com/role: crossplane
  vault.hashicorp.com/agent-run-as-user: "65532"
EOF
```

接下来，将 Crossplane ESS Plugin pod 安装到 `crossplane-system` 名称空间，并应用 Vault 注释。

```shell
helm upgrade --install ess-plugin-vault oci://xpkg.upbound.io/crossplane-contrib/ess-plugin-vault --namespace crossplane-system -f values.yaml
```

## 配置 crossplane

使用 Vault 插件需要配置才能连接到 Vault 服务。 该插件还需要 Provider 启用外部存储。

配置好插件和 Provider 后，Crossplane 需要两个 `StoreConfig` 对象来描述 Crossplane 和 Provider 如何与 Vault 通信。

### 在 Provider 中启用外部secret存储

{{<hint "note">}}本例被引用的是 Provider GCP，但{{<hover label="ControllerConfig" line="2">}}控制器配置{{</hover>}}对所有 Provider 都一样。{{</hint >}}

创建一个 `ControllerConfig` 对象，以启用外部secret存储。

```yaml {label="ControllerConfig"}
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: vault-config
spec:
  args:
    - --enable-external-secret-stores" | kubectl apply -f -
```

安装 Provider 并应用 ControllerConfig。

```yaml
echo "apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.23.0-rc.0.19.ge9b75ee5
  controllerConfigRef:
    name: vault-config" | kubectl apply -f -
```

###将 crossplane 插件连接到 Vault

创建一个 {{<hover label="VaultConfig" line="2">}}VaultConfig{{</hover>}}资源，以便插件连接到 Vault 服务: 

```yaml {label="VaultConfig"}
echo "apiVersion: secrets.crossplane.io/v1alpha1
kind: VaultConfig
metadata:
  name: vault-internal
spec:
  server: http://vault.vault-system:8200
  mountPath: secret/
  version: v2
  auth:
    method: Token
    token:
      source: Filesystem
      fs:
        path: /vault/secrets/token" | kubectl apply -f -
```

### 创建一个 crossplane StoreConfig

创建一个 {{<hover label="xp-storeconfig" line="2">}}存储配置{{</hover >}}对象创建一个{{<hover label="xp-storeconfig" line="1">}}secrets.crossplane.io 中创建一个 StoreConfig 对象。{{</hover >}}crossplane 使用 StoreConfig 连接到 Vault 插件服务。

配置 {{<hover label="xp-storeconfig" line="10">}}配置参考{{</hover >}}将 StoreConfig 连接到特定的 Vault 插件配置。

```yaml {label="xp-storeconfig"}
echo "apiVersion: secrets.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
spec:
  type: Plugin
  defaultScope: crossplane-system
  plugin:
    endpoint: ess-plugin-vault.crossplane-system:4040
    configRef:
      apiVersion: secrets.crossplane.io/v1alpha1
      kind: VaultConfig
      name: vault-internal" | kubectl apply -f -
```

### 创建 Provider StoreConfig

创建一个 {{<hover label="gcp-storeconfig" line="2">}}存储配置{{</hover >}}对象、{{<hover label="gcp-storeconfig" line="1">}}提供程序使用该 StoreConfig 与 Vault 进行通信。{{</hover >}}Provider 使用该 StoreConfig 与 Vault 通信，以获取托管资源。

配置 {{<hover label="gcp-storeconfig" line="10">}}配置参考{{</hover >}}将 StoreConfig 连接到特定的 Vault 插件配置。

```yaml {label="gcp-storeconfig"}
echo "apiVersion: gcp.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
spec:
  type: Plugin
  defaultScope: crossplane-system
  plugin:
    endpoint: ess-plugin-vault.crossplane-system:4040
    configRef:
      apiVersion: secrets.crossplane.io/v1alpha1
      kind: VaultConfig
      name: vault-internal" | kubectl apply -f -
```

## 创建 Provider 资源

检查 crossplane 是否安装了 Provider，Provider 是否健康。

```shell {copy-lines="1"}
kubectl get providers
NAME INSTALLED HEALTHY PACKAGE AGE
provider-gcp True True xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.23.0-rc.0.19.ge9b75ee5 10m
```

### 创建复合资源定义

创建一个 `CompositeResourceDefinition` 来定义自定义 API 端点。

```yaml
echo "apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: compositeessinstances.ess.example.org
  annotations:
    feature: ess
spec:
  group: ess.example.org
  names:
    kind: CompositeESSInstance
    plural: compositeessinstances
  claimNames:
    kind: ESSInstance
    plural: essinstances
  connectionSecretKeys:
    - publicKey
    - publicKeyType
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
              parameters:
                type: object
                properties:
                  serviceAccount:
                    type: string
                required:
                  - serviceAccount
            required:
              - parameters" | kubectl apply -f -
```

#### 创建一个构图

创建一个 "Composition"，以便在 GCP 内创建服务账户和服务账户密钥。

创建服务帐户密钥{{<hover label="comp" line="39" >}}连接详情{{</hover>}}并存储在 Vault 中，Provider 会使用被引用的{{<hover label="comp" line="31">}}发布连接详情到{{</hover>}}详细信息。

```yaml {label="comp"}
echo "apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: essinstances.ess.example.org
  labels:
    feature: ess
spec:
  publishConnectionDetailsWithStoreConfigRef: 
    name: vault
  compositeTypeRef:
    apiVersion: ess.example.org/v1alpha1
    kind: CompositeESSInstance
  resources:
    - name: serviceaccount
      base:
        apiVersion: iam.gcp.crossplane.io/v1alpha1
        kind: ServiceAccount
        metadata:
          name: ess-test-sa
        spec:
          forProvider:
            displayName: a service account to test ess
    - name: serviceaccountkey
      base:
        apiVersion: iam.gcp.crossplane.io/v1alpha1
        kind: ServiceAccountKey
        spec:
          forProvider:
            serviceAccountSelector:
              matchControllerRef: true
          publishConnectionDetailsTo:
            name: ess-mr-conn
            metadata:
              labels:
                environment: development
                team: backend
            configRef:
              name: vault
      connectionDetails:
        - fromConnectionSecretKey: publicKey
        - fromConnectionSecretKey: publicKeyType" | kubectl apply -f -
```

### 创建claim

现在创建一个 "声明"，让 crossplane 创建 GCP 资源和相关secret。

与 Composition 一样，Claim 也被引用为{{<hover label="claim" line="12">}}发布连接详情到{{</hover>}}连接到 Vault 并存储secret。

```yaml {label="claim"}
echo "apiVersion: ess.example.org/v1alpha1
kind: ESSInstance
metadata:
  name: my-ess
  namespace: default
spec:
  parameters:
    serviceAccount: ess-test-sa
  compositionSelector:
    matchLabels:
      feature: ess
  publishConnectionDetailsTo:
    name: ess-claim-conn
    metadata:
      labels:
        environment: development
        team: backend
    configRef:
      name: vault" | kubectl apply -f -
```

## 验证资源

验证所有资源是否 "READY "和 "SYNCED": 

```shell {copy-lines="1"}
kubectl get managed
NAME READY SYNCED DISPLAYNAME EMAIL DISABLED
serviceaccount.iam.gcp.crossplane.io/my-ess-zvmkz-vhklg True True a service account to test ess my-ess-zvmkz-vhklg@testingforbugbounty.iam.gserviceaccount.com

NAME READY SYNCED KEY_ID CREATED_AT EXPIRES_AT
serviceaccountkey.iam.gcp.crossplane.io/my-ess-zvmkz-bq8pz True True 5cda49b7c32393254b5abb121b4adc07e140502c 2022-03-23T10:54:50Z
```

查看claim

```shell {copy-lines="1"}
kubectl -n default get claim
NAME READY CONNECTION-SECRET AGE
my-ess True 19s
```

查看 Composition 资源。

```shell {copy-lines="1"}
kubectl get composite
NAME READY COMPOSITION AGE
my-ess-zvmkz True essinstances.ess.example.org 32s
```

## 验证 Vault secret

查看 Vault 内部，查看来自管理资源的secret。

```shell {copy-lines="1",label="vault-key"}
kubectl -n vault-system exec -i vault-0 -- vault kv list /secret/default
Keys
----
ess-claim-conn
```

关键字 {{<hover label="vault-key" line="4">}}ess-claim-conn{{</hover>}}是claim的{{<hover label="claim" line="12">}}publishConnectionDetailsTo{{</hover>}}配置的名称。

检查 "crossplane-system "Vault 范围内的连接secret。

```shell {copy-lines="1",label="scope-key"}
kubectl -n vault-system exec -i vault-0 -- vault kv list /secret/crossplane-system
Keys
----
d2408335-eb88-4146-927b-8025f405da86
ess-mr-conn
```

钥匙{{<hover label="scope-key"line="4">}}d2408335-eb88-4146-927b-8025f405da86{{</hover>}}来自

<!-- ## WHERE DOES IT COME FROM? -->

和钥匙{{<hover label="scope-key"line="5">}}本质-先生-康{{</hover>}}来自 Composition 的{{<hover label="comp" line="31">}}publishConnectionDetailsTo{{</hover>}}配置。

检查 claims 的连接secret `ess-claim-conn` 的内容，查看管理资源创建的密钥。

```shell {copy-lines="1"}
kubectl -n vault-system exec -i vault-0 -- vault kv get /secret/default/ess-claim-conn
======= Metadata =======
Key Value
---                -----
created_time 2022-03-18T21:24:07.2085726Z
custom_metadata map[environment:development secret.crossplane.io/ner-uid:881cd9a0-6cc6-418f-8e1d-b36062c1e108 team:backend]
deletion_time n/a
destroyed false
version 1

======== Data ========
Key Value
---              -----
publicKey        -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzsEYCokmYEsZJCc9QN/8
Fm1M/kTPp7Gat/MXLTP3zFyCTBFVNLN79MbAKdinWi6ePXEb75vzB79IdZcWj8lo
8trnS64QjNB9Vs4Xk5UvDALwleFN/bZeperxivDPwVPvT9Aqy/U9kohoS/LHyE8w
uWQb5AuMeVQ1gtCTnCqQZ4d2MSVhQXYVvAWax1spJ9LT7mHub5j95xDdYIcOV3VJ
l9CIo4VrWIT8THFN2NnjTrGq9+0TzXY0bV674bjJkfBC6v6yXs5HTetG+Uekq/xf
FCjrrDi1+2UR9Mu2WTuvl8qn50be+mbwdJO5wE32jewxdYrVVmj19+PkaEeAwGTc
vwIDAQAB
-----END PUBLIC KEY-----
publicKeyType TYPE_RAW_PUBLIC_KEY
```

检查托管资源连接secret `ess-mr-conn` 的内容。 公钥与 claim 中的公钥相同，因为 claim 正在引用此托管资源。

```shell {copy-lines="1"}
kubectl -n vault-system exec -i vault-0 -- vault kv get /secret/crossplane-system/ess-mr-conn
======= Metadata =======
Key Value
---                -----
created_time 2022-03-18T21:21:07.9298076Z
custom_metadata map[environment:development secret.crossplane.io/ner-uid:4cd973f8-76fc-45d6-ad45-0b27b5e9252a team:backend]
deletion_time n/a
destroyed false
version 2

========= Data =========
Key Value
---               -----
privateKey        {
  "type": "service_account",
  "project_id": "REDACTED",
  "private_key_id": "REDACTED",
  "private_key": "-----BEGIN PRIVATE KEY-----\nREDACTED\n-----END PRIVATE KEY-----\n",
  "client_email": "ess-test-sa@REDACTED.iam.gserviceaccount.com",
  "client_id": "REDACTED",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/ess-test-sa%40REDACTED.iam.gserviceaccount.com"
}
privateKeyType TYPE_GOOGLE_CREDENTIALS_FILE
publicKey         -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzsEYCokmYEsZJCc9QN/8
Fm1M/kTPp7Gat/MXLTP3zFyCTBFVNLN79MbAKdinWi6ePXEb75vzB79IdZcWj8lo
8trnS64QjNB9Vs4Xk5UvDALwleFN/bZeperxivDPwVPvT9Aqy/U9kohoS/LHyE8w
uWQb5AuMeVQ1gtCTnCqQZ4d2MSVhQXYVvAWax1spJ9LT7mHub5j95xDdYIcOV3VJ
l9CIo4VrWIT8THFN2NnjTrGq9+0TzXY0bV674bjJkfBC6v6yXs5HTetG+Uekq/xf
FCjrrDi1+2UR9Mu2WTuvl8qn50be+mbwdJO5wE32jewxdYrVVmj19+PkaEeAwGTc
vwIDAQAB
-----END PUBLIC KEY-----
publicKeyType TYPE_RAW_PUBLIC_KEY
```

#### 删除资源

删除claim会从 Vault 中删除受管资源和相关密钥。

```shell
kubectl delete claim my-ess
```

<!-- named links -->
