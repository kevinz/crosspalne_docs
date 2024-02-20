---

title: 用 Python 编写一个组成函数
状态: beta
alphaVersion: "1.11"
betaVersion: "1.14"
weight: 81
description: "通过 Composition 函数，您可以使用 Python 将资源模板化"

---

Composition 函数（简称函数）是模板化 Crossplane 资源的自定义程序。 当你创建复合资源 (XR) 时，Crossplane 会调用 Composition 函数来决定它应该创建哪些资源。阅读 [concepts](https://docs.crossplane.io/latest/concepts/composition-functions) 页面了解更多有关 Composition 函数的信息。

您可以使用通用编程语言为模板资源编写函数。 使用通用编程语言可以让函数对模板资源使用高级逻辑，如循环和条件。 本指南介绍如何在 [Python](https://python.org) 中编写 Composition 函数。

{{< hint "important" >}}在阅读本指南之前，最好先熟悉一下 [Composition 功能的工作原理](https://docs.crossplane.io/latest/concepts/composition-functions#how-composition-functions-work)。{{< /hint >}}

## 了解步骤

本指南介绍为{{<hover label="xr" line="2">}}XBuckets{{</hover>}}Composition 资源 (XR) 的合成函数。

```yaml {label="xr"}
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
spec:
  region: us-east-2
  names:
  - crossplane-functions-example-a
  - crossplane-functions-example-b
  - crossplane-functions-example-c
```

<!-- vale gitlab.FutureTense = NO -->

<!--
This section is setting the stage for future sections. It doesn't make sense to
refer to the function in the present tense, because it doesn't exist yet.
-->

一个 `XBuckets` XR 有一个区域和一个桶名数组。 该函数将为名称数组中的每个条目创建一个亚马逊网络服务（AWS）S3 桶。

<!-- vale gitlab.FutureTense = YES -->

用 Python 写一个函数

1.[安装编写函数所需的工具](#install-the-tools-you-need-to-write-the-function)
2.[从模板初始化函数](#initialize-the-function-from-a-template)
3.[编辑模板，添加函数逻辑](#edit-the-template-to-add-the-functions-logic)
4.[测试端到端函数](#test-the-function-end-to-end)
5.[构建函数并将其推送到软件包注册库](#build-and-push-the-function-to-a-package-registry)

本指南将详细介绍每个步骤。

## 安装编写函数所需的工具

要在 Python 中编写一个函数，您需要

* [Python](https://www.python.org/downloads/) v3.11。
* [Hatch](https://hatch.pypa.io/)，一个 Python 构建工具。本指南被引用 v1.7。
* [Docker Engine](https://docs.docker.com/engine/)。本指南被引用 Engine v24。
* Crossplane CLI](https://docs.crossplane.io/latest/cli) v1.14 或更新版本。本指南被引用的是 Crossplane CLI v1.14。

{{<hint "note">}}你不需要访问 Kubernetes 集群或 crossplane 控制平面，就能构建或测试 Composition 功能。{{</hint>}}

## 从模板初始化函数

使用 "crossplane beta xpkg init "命令初始化一个新函数。运行该命令时，它会以[一个 GitHub 仓库](https://github.com/crossplane/function-template-python)为模板初始化你的函数。

```shell {copy-lines=1}
crossplane beta xpkg init function-xbuckets https://github.com/crossplane/function-template-python -d function-xbuckets
Initialized package "function-xbuckets" in directory "/home/negz/control/negz/function-xbuckets" from https://github.com/crossplane/function-template-python/tree/bfed6923ab4c8e7adeed70f41138645fc7d38111 (main)
```

使用 `crossplane beta init xpkg` 命令会创建一个名为 `function-xbuckets` 的目录。 运行该命令后，新目录应如下所示: 

```shell {copy-lines=1}
ls function-xbuckets
Dockerfile example/  function/  LICENSE package/  pyproject.toml README.md renovate.json tests/
```

您的函数代码位于 `function` 目录中: 

```shell {copy-lines=1}
ls function/
__version__.py fn.py main.py
```

`function/fn.py` 文件是添加函数代码的地方。 了解模板中的其他一些文件很有用: 

* `function/main.py` 运行函数。你不需要编辑 `main.py`。
* `Dockerfile` 运行函数。不需要编辑 `Dockerfile`。
* package` 目录包含被引用用于构建函数包的元数据。

{{<hint "tip">}}

<!-- vale gitlab.FutureTense = NO -->

<!--
This tip talks about future plans for Crossplane.
-->

在 Crossplane CLI v1.14 中，"crossplane beta xpkg init "只是克隆了一个模板 GitHub 仓库。 未来的 CLI 发布将自动执行用新函数名称替换模板名称等任务。 详情请参见 Crossplane 问题 [#4941](https://github.com/crossplane/crossplane/issues/4941)。

<!-- vale gitlab.FutureTense = YES -->

{{</hint>}}

在开始添加代码之前，编辑 `package/crossplane.yaml` 更改软件包名称。 将软件包命名为 `function-xbuckets`。

package/input 目录定义了函数输入的 OpenAPI 模式。 本指南中的函数不接受输入。 删除 `package/input` 目录。

组成函数](https://docs.crossplane.io/latest/concepts/composition-functions) 文档解释了组成函数的输入。

{{<hint "tip">}}如果您正在编写一个被引用的函数，请编辑输入 YAML 文件以满足您的函数要求。

更改输入的种类和 API group。 不要使用 "Input "和 "template.fn.crossplane.io"，而应使用对函数有意义的名称。{{</hint>}}

## 编辑模板，添加函数逻辑

您可以在{{<hover label="hello-world" line="1">}}运行函数{{</hover>}}方法中添加您的函数逻辑。 首次打开该文件时，它包含一个 "hello world "函数。

```python {label="hello-world"}
async def RunFunction(self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext) -> fnv1beta1.RunFunctionResponse:
    log = self.log.bind(tag=req.meta.tag)
    log.info("Running function")

    rsp = response.to(req)

    example = ""
    if "example" in req.input:
        example = req.input["example"]

    # TODO: Add your function logic here!
    response.normal(rsp, f"I was run with input {example}!")
    log.info("I was run!", input=example)

    return rsp
```

所有 Python Composition 函数都有一个 "RunFunction "方法。 crossplane 会在一个{{<hover label="hello-world" line="1">}}RunFunctionRequest{{</hover>}}对象中传递函数运行所需的一切。

该函数通过返回一个{{<hover label="hello-world" line="15">}}运行函数响应{{</hover>}}对象。

编辑 `RunFunction` 方法，将其替换为以下代码。

```python {hl_lines="7-28"}
async def RunFunction(self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext) -> fnv1beta1.RunFunctionResponse:
    log = self.log.bind(tag=req.meta.tag)
    log.info("Running function")

    rsp = response.to(req)

    region = req.observed.composite.resource["spec"]["region"]
    names = req.observed.composite.resource["spec"]["names"]

    for name in names:
        rsp.desired.resources[f"xbuckets-{name}"].resource.update(
            {
                "apiVersion": "s3.aws.upbound.io/v1beta1",
                "kind": "Bucket",
                "metadata": {
                    "annotations": {
                        "crossplane.io/external-name": name,
                    },
                },
                "spec": {
                    "forProvider": {
                        "region": region,
                    },
                },
            }
        )

    log.info("Added desired buckets", region=region, count=len(names))

    return rsp
```

展开下面的代码块以查看完整的 `fn.py`，包括导入和解释函数逻辑的注释。

{{<expand "The full fn.py file" >}}

```python
"""A Crossplane composition function."""

import grpc
from crossplane.function import logging, response
from crossplane.function.proto.v1beta1 import run_function_pb2 as fnv1beta1
from crossplane.function.proto.v1beta1 import run_function_pb2_grpc as grpcv1beta1

class FunctionRunner(grpcv1beta1.FunctionRunnerService):
    """A FunctionRunner handles gRPC RunFunctionRequests."""

    def __init__(self):
        """Create a new FunctionRunner."""
        self.log = logging.get_logger()

    async def RunFunction(
        self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext
    ) -> fnv1beta1.RunFunctionResponse:
        """Run the function."""
        # Create a logger for this request.
        log = self.log.bind(tag=req.meta.tag)
        log.info("Running function")

        # Create a response to the request. This copies the desired state and
        # pipeline context from the request to the response.
        rsp = response.to(req)

        # Get the region and a list of bucket names from the observed composite
        # resource (XR). Crossplane represents resources using the Struct
        # well-known protobuf type. The Struct Python object can be accessed
        # like a dictionary.
        region = req.observed.composite.resource["spec"]["region"]
        names = req.observed.composite.resource["spec"]["names"]

        # Add a desired S3 bucket for each name.
        for name in names:
            # Crossplane represents desired composed resources using a protobuf
            # map of messages. This works a little like a Python defaultdict.
            # Instead of assigning to a new key in the dict-like map, you access
            # the key and mutate its value as if it did exist.
            #
            # The below code works because accessing the xbuckets-{name} key
            # automatically creates a new, empty fnv1beta1.Resource message. The
            # Resource message has a resource field containing an empty Struct
            # object that can be populated from a dictionary by calling update.
            #
            # https://protobuf.dev/reference/python/python-generated/#map-fields
            rsp.desired.resources[f"xbuckets-{name}"].resource.update(
                {
                    "apiVersion": "s3.aws.upbound.io/v1beta1",
                    "kind": "Bucket",
                    "metadata": {
                        "annotations": {
                            "crossplane.io/external-name": name,
                        },
                    },
                    "spec": {
                        "forProvider": {
                            "region": region,
                        },
                    },
                }
            )

        # Log what the function did. This will only appear in the function's pod
        # logs. A function can use response.normal() and response.warning() to
        # emit Kubernetes events associated with the XR it's operating on.
        log.info("Added desired buckets", region=region, count=len(names))

        return rsp
```

{{</expand>}}

此代码

1.从 `RunFunctionRequest` 获取观察到的 Composition 资源。
2.从观察到的 Composition 资源中获取区域和水桶名称。
3.为每个桶名添加一个所需的 S3 桶。
4.在 "RunFunctionResponse "中返回所需的 S3 存储桶。

crossplane 提供了一个 [软件开发工具包](https://github.com/crossplane/function-sdk-python) (SDK)，用于用 Python 编写 Composition 函数。本函数被引用了 SDK 中的实用工具。

{{<hint "tip">}}请阅读 [Python 函数 SDK 文档](https://crossplane.github.io/function-sdk-python)。{{</hint>}}

{{<hint "important">}}Python SDK 会根据 [Protocol Buffers](https://protobuf.dev) 模式自动生成 `RunFunctionRequest` 和 `RunFunctionResponse` Python 对象。您可以在 [Buf Schema Registry](https://buf.build/crossplane/crossplane/docs/main:apiextensions.fn.proto.v1beta1) 中查看该模式。

生成的 Python 对象的字段与内置的 Python 类型（如字典和列表）的行为类似。 请注意，它们之间存在一些差异。

值得注意的是，您可以像访问字典一样访问观察到的资源和所需资源的映射，但不能通过为映射键赋值来添加新的所需资源。 相反，访问和 mutation 映射键就好像它已经存在一样。

而不是像这样添加一个新资源: 

```python
resource = {"apiVersion": "example.org/v1", "kind": "Composed", ...}
rsp.desired.resources["new-resource"] = fnv1beta1.Resource(resource=resource)
```

假装它已经存在，然后对它进行 mutation，就像这样: 

```python
resource = {"apiVersion": "example.org/v1", "kind": "Composed", ...}
rsp.desired.resources["new-resource"].resource.update(resource)
```

更多详情，请参阅协议缓冲区 [Python 生成代码指南](https://protobuf.dev/reference/python/python-generated/#fields)。{{</hint>}}

## 测试端到端功能

通过添加单元测试和被引用 "crossplane beta render "命令来测试你的函数。

当你从模板初始化一个函数时，它会在 `tests/test_fn.py` 中添加一些单元测试。这些测试使用 Python 标准库中的 [`unittest`](https://docs.python.org/3/library/unittest.html) 模块。

要添加测试用例，请更新 `test_run_function` 中的 `cases` 列表。 展开下面的代码块，查看函数的完整 `tests/test_fn.py` 文件。

{{<expand "The full test_fn.py file" >}}

```python
import dataclasses
import unittest

from crossplane.function import logging, resource
from crossplane.function.proto.v1beta1 import run_function_pb2 as fnv1beta1
from google.protobuf import duration_pb2 as durationpb
from google.protobuf import json_format
from google.protobuf import struct_pb2 as structpb

from function import fn

class TestFunctionRunner(unittest.IsolatedAsyncioTestCase):
    def setUp(self) -> None:
        logging.configure(level=logging.Level.DISABLED)
        self.maxDiff = 2000

    async def test_run_function(self) -> None:
        @dataclasses.dataclass
        class TestCase:
            reason: str
            req: fnv1beta1.RunFunctionRequest
            want: fnv1beta1.RunFunctionResponse

        cases = [
            TestCase(
                reason="The function should compose two S3 buckets.",
                req=fnv1beta1.RunFunctionRequest(
                    observed=fnv1beta1.State(
                        composite=fnv1beta1.Resource(
                            resource=resource.dict_to_struct(
                                {
                                    "apiVersion": "example.crossplane.io/v1alpha1",
                                    "kind": "XBuckets",
                                    "metadata": {"name": "test"},
                                    "spec": {
                                        "region": "us-east-2",
                                        "names": ["test-bucket-a", "test-bucket-b"],
                                    },
                                }
                            )
                        )
                    )
                ),
                want=fnv1beta1.RunFunctionResponse(
                    meta=fnv1beta1.ResponseMeta(ttl=durationpb.Duration(seconds=60)),
                    desired=fnv1beta1.State(
                        resources={
                            "xbuckets-test-bucket-a": fnv1beta1.Resource(
                                resource=resource.dict_to_struct(
                                    {
                                        "apiVersion": "s3.aws.upbound.io/v1beta1",
                                        "kind": "Bucket",
                                        "metadata": {
                                            "annotations": {
                                                "crossplane.io/external-name": "test-bucket-a"
                                            },
                                        },
                                        "spec": {
                                            "forProvider": {"region": "us-east-2"}
                                        },
                                    }
                                )
                            ),
                            "xbuckets-test-bucket-b": fnv1beta1.Resource(
                                resource=resource.dict_to_struct(
                                    {
                                        "apiVersion": "s3.aws.upbound.io/v1beta1",
                                        "kind": "Bucket",
                                        "metadata": {
                                            "annotations": {
                                                "crossplane.io/external-name": "test-bucket-b"
                                            },
                                        },
                                        "spec": {
                                            "forProvider": {"region": "us-east-2"}
                                        },
                                    }
                                )
                            ),
                        },
                    ),
                    context=structpb.Struct(),
                ),
            ),
        ]

        runner = fn.FunctionRunner()

        for case in cases:
            got = await runner.RunFunction(case.req, None)
            self.assertEqual(
                json_format.MessageToDict(got),
                json_format.MessageToDict(case.want),
                "-want, +got",
            )

if __name__ == "__main__":
    unittest.main()
```

{{</expand>}}

使用 `hatch run` 运行单元测试: 

```shell {copy-lines="1"}
hatch run test:unit
.
----------------------------------------------------------------------
Ran 1 test in 0.003s

OK
```

{{<hint "tip">}}[Hatch](https://hatch.pypa.io/)是一个 Python 构建工具。它可以构建类似 wheels 的 Python 构件。它还可以管理虚拟环境，类似于 `virtualenv` 或 `venv`。`hatch run` 命令会创建一个虚拟环境，并在该环境中运行命令。{{</hint>}}

您可以使用 Crossplane CLI 预览被引用此功能的 Composition 的 Output，不需要使用 Crossplane 控制平面就能完成此操作。

在 `function-xbuckets` 下创建名为 `example` 的目录，并创建 Composite Resource、Composition 和 Function YAML 文件。

展开以下区块，查看示例文件。

{{<expand "The xr.yaml, composition.yaml and function.yaml files">}}

您可以使用这些文件，通过运行 `crossplane beta render` 重现下面的输出结果。

XR.yaml` 文件包含要渲染的 Composition 资源: 

```yaml
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
spec:
  region: us-east-2
  names:
  - crossplane-functions-example-a
  - crossplane-functions-example-b
  - crossplane-functions-example-c
```

<br />

composition.yaml` 文件包含用于渲染复合资源的 Composition: 

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: create-buckets
spec:
  compositeTypeRef:
    apiVersion: example.crossplane.io/v1
    kind: XBuckets
  mode: Pipeline
  pipeline:
  - step: create-buckets
    functionRef:
      name: function-xbuckets
```

<br />

functions.yaml` 文件包含 Composition 在其 Pipelines 步骤中引用的函数: 

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-xbuckets
  annotations:
    render.crossplane.io/runtime: Development
spec:
  # The CLI ignores this package when using the Development runtime.
  # You can set it to any value.
  package: xpkg.upbound.io/negz/function-xbuckets:v0.1.0
```

{{</expand>}}

functions.yaml` 中的函数被引用为{{<hover label="development" line="6">}}开发{{</hover>}}运行时。 这会告诉 `crossplane beta render` 您的函数正在本地运行。 它会连接到您本地运行的函数，而不是被引用 Docker 来拉动和运行函数。

```yaml {label="development"}
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-xbuckets
  annotations:
    render.crossplane.io/runtime: Development
```

使用 `hatch run development` 在本地运行您的函数。

```shell {label="run"}
hatch run development
```

{{<hint "warning">}}`hatch run development` 在不进行加密或身份验证的情况下运行函数。 仅在测试和开发过程中使用。{{</hint>}}

在另一个终端中，运行 `crossplane beta render`。

```shell
crossplane beta render xr.yaml composition.yaml functions.yaml
```

该命令调用你的函数。 在运行函数的终端中，现在应该可以看到 logging 输出: 

```shell
hatch run development
2024-01-11T22:12:58.153572Z [info     ] Running function filename=fn.py lineno=22 tag=
2024-01-11T22:12:58.153792Z [info     ] Added desired buckets count=3 filename=fn.py lineno=68 region=us-east-2 tag=
```

crossplane beta render` 命令会打印函数返回的所需资源。

```yaml
---
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-b
    crossplane.io/external-name: crossplane-functions-example-b
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-c
    crossplane.io/external-name: crossplane-functions-example-c
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-a
    crossplane.io/external-name: crossplane-functions-example-a
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
```

{{<hint "tip">}}请阅读组成函数文档，了解有关 [测试组成函数](https://docs.crossplane.io/latest/concepts/composition-functions#test-a-composition-that-uses-functions) 的更多信息。{{</hint>}}

## 构建函数并将其推送至 packages 注册表

构建函数分为两个阶段: 首先是构建函数的运行时，这是 Crossplane 用来运行函数的开放容器倡议（OCI）镜像。 然后将运行时嵌入软件包，并将其推送到软件包注册中心。 Crossplane CLI 将 `xpkg.upbound.io` 作为默认的软件包注册中心。

一个函数默认支持单个平台，如 "linux/amd64"，您可以为每个平台构建运行时和软件包，然后将所有软件包推送到注册表中的单个标签，从而支持多个平台。

将您的函数推送到 registry，就可以在 crossplane 控制平面中使用您的函数。请参阅[Composition functions documentation](https://docs.crossplane.io/latest/concepts/composition-functions)。了解如何在控制平面中使用函数。

使用 docker 为每个平台构建运行时。

```shell {copy-lines="1"}
docker build . --quiet --platform=linux/amd64 --tag runtime-amd64
sha256:fdf40374cc6f0b46191499fbc1dbbb05ddb76aca854f69f2912e580cfe624b4b
```

```shell {copy-lines="1"}
docker build . --quiet --platform=linux/arm64 --tag runtime-arm64
sha256:cb015ceabf46d2a55ccaeebb11db5659a2fb5e93de36713364efcf6d699069af
```

{{<hint "tip">}}您可以使用任何标签，无需将运行时镜像推送到 registry。 标签只是用来告诉 `crossplane xpkg build` 嵌入什么运行时。{{</hint>}}

{{<hint "important">}}Docker 使用仿真技术为不同平台创建镜像。 如果为不同平台构建镜像失败，请确保已安装 `binfmt`。有关说明，请参阅 [Docker 文档](https://docs.docker.com/build/building/multi-platform/#qemu)。{{</hint>}}

使用 Crossplane CLI 为每个平台构建一个软件包。 每个软件包都嵌入了一个运行时镜像。

......。 {{<hover label="build" line="2">}}--package-root{{</hover>}}flag 指定了包含 `crossplane.yaml` 的 `package` 目录。 其中包括软件包的元数据。

......。 {{<hover label="build" line="3">}}--嵌入运行时镜像{{</hover>}}flag 指定了被引用 Docker 构建的运行时镜像标签。

在 {{<hover label="build" line="4">}}--package-file{{</hover>}}标志指定将软件包文件写入磁盘的位置。 crossplane 软件包文件的扩展名为 `.xpkg`。

```shell {label="build"}
crossplane xpkg build \
    --package-root=package \
    --embed-runtime-image=runtime-amd64 \
    --package-file=function-amd64.xpkg
```

```shell
crossplane xpkg build \
    --package-root=package \
    --embed-runtime-image=runtime-arm64 \
    --package-file=function-arm64.xpkg
```

{{<hint "tip">}}crossplane 软件包是特殊的 OCI 镜像。请在【软件包文档】(https://docs.crossplane.io/latest/concepts/packages) 中阅读有关软件包的更多信息。{{</hint>}}

将两个软件包文件都推送到注册表中。将两个文件都推送到注册表中的一个标签，就能创建一个[多平台](https://docs.docker.com/build/building/multi-platform/) 软件包，在 `linux/arm64` 和 `linux/amd64` 主机上都能运行。

```shell
crossplane xpkg push \
  --package-files=function-amd64.xpkg,function-arm64.xpkg \
  negz/function-xbuckets:v0.1.0
```

{{<hint "tip">}}如果您将函数推送到 GitHub 仓库，模板会使用 [GitHub Actions](https://github.com/features/actions) 自动设置持续集成 (CI)。CI 工作流将对您的函数进行校验、测试和构建。您可以通过阅读 `.github/workflows/ci.yaml`，查看模板是如何配置 CI 的。

CI 工作流可以自动将软件包推送到 `xpkg.upbound.io`。要做到这一点，您必须在 https://marketplace.upbound.io 创建一个版本库。通过创建一个 API 令牌并[将其添加到您的版本库](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository)，赋予 CI 工作流向市场推送的权限。将您的 API 令牌访问 ID 保存为名为 `XPKG_ACCESS_ID` 的secret，并将您的 API 令牌保存为名为 `XPKG_TOKEN` 的secret。{{</hint>}}