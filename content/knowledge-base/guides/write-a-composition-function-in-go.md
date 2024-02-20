---

title: 用 Go 编写一个 Composition 函数
状态: beta
alphaVersion: "1.11"
betaVersion: "1.14"
weight: 80
description: "通过 Composition 函数，您可以使用 Go 将资源模板化"

---

Composition 函数（简称函数）是模板化 Crossplane 资源的自定义程序。 当你创建复合资源 (XR) 时，Crossplane 会调用 Composition 函数来决定它应该创建哪些资源。阅读 [concepts](https://docs.crossplane.io/latest/concepts/composition-functions) 页面了解更多有关 Composition 函数的信息。

您可以使用通用编程语言为模板资源编写函数。 使用通用编程语言可以让函数对模板资源使用高级逻辑，如循环和条件。 本指南介绍如何使用 [Go](https://go.dev)编写 Composition 函数。

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

用 Go 编写函数

1.[安装编写函数所需的工具](#install-the-tools-you-need-to-write-the-function)
2.[从模板初始化函数](#initialize-the-function-from-a-template)
3.[编辑模板，添加函数逻辑](#edit-the-template-to-add-the-functions-logic)
4.[测试端到端函数](#test-the-function-end-to-end)
5.[构建函数并将其推送到软件包注册库](#build-and-push-the-function-to-a-package-registry)

本指南将详细介绍每个步骤。

## 安装编写函数所需的工具

用 Go 编写函数需要

* [Go](https://go.dev/dl/) v1.21 或更新版本。本指南被引用 Go v1.21。
* [Docker Engine](https://docs.docker.com/engine/).本指南被引用 Engine v24。
* Crossplane CLI](https://docs.crossplane.io/latest/cli) v1.14 或更新版本。本指南被引用 Crossplane CLI v1.14。

{{<hint "note">}}你不需要访问 Kubernetes 集群或 crossplane 控制平面，就能构建或测试 Composition 功能。{{</hint>}}

## 从模板初始化函数

使用 "crossplane beta xpkg init "命令初始化一个新函数。运行该命令时，它会以[一个 GitHub 仓库](https://github.com/crossplane/function-template-go)为模板初始化你的函数。

```shell {copy-lines=1}
crossplane beta xpkg init function-xbuckets function-template-go -d function-xbuckets 
Initialized package "function-xbuckets" in directory "/home/negz/control/negz/function-xbuckets" from https://github.com/crossplane/function-template-go/tree/91a1a5eed21964ff98966d72cc6db6f089ad63f4 (main)
```

使用 `crossplane beta init xpkg` 命令会创建一个名为 `function-xbuckets` 的目录。 运行该命令后，新目录应如下所示: 

```shell {copy-lines=1}
ls function-xbuckets
Dockerfile fn.go fn_test.go go.mod go.sum input/  LICENSE main.go package/  README.md renovate.json
```

fn.go "文件是添加函数代码的地方，了解模板中的其他一些文件非常有用: 

* `main.go` 运行函数。你不需要编辑 `main.go`。
* `Dockerfile` 运行函数。不需要编辑 `Dockerfile`。
* `input` 目录定义了函数的输入类型。
* package 目录包含被引用用于构建函数包的元数据。

{{<hint "tip">}}

<!-- vale gitlab.FutureTense = NO -->

<!--
This tip talks about future plans for Crossplane.
-->

在 Crossplane CLI v1.14 中，"crossplane beta xpkg init "只是克隆了一个模板 GitHub 仓库。 未来的 CLI 发布将自动执行用新函数名称替换模板名称等任务。 详情请参见 Crossplane 问题 [#4941](https://github.com/crossplane/crossplane/issues/4941)。

<!-- vale gitlab.FutureTense = YES -->

{{</hint>}}

在开始添加代码之前，您必须进行一些更改: 

* 编辑 `package/crossplane.yaml` 更改 packages 的名称。
* 编辑 `go.mod` 更改 Go 模块的名称。

将软件包命名为 `function-xbuckets`.

模块的名称取决于你想把函数代码保存在哪里。 如果你把 Go 代码推送到 GitHub，可以使用你的 GitHub 用户名。 例如 `module github.com/negz/function-xbuckets`。

本指南中的函数不引用输入类型。 对于该函数，应删除 `input` 和 `package/input` 目录。

`input` 目录定义了一个 Go 结构，函数可以使用该结构从 Composition 中获取输入。[Composition functions](https://docs.crossplane.io/latest/concepts/composition-functions) 文档解释了如何将输入传递给 Composition 函数。

package/input "目录包含由 "input "目录中的结构生成的 OpenAPI 模式。

{{<hint "tip">}}如果您正在编写一个被引用的函数，请对输入进行编辑，以满足您的函数要求。

更改输入的种类和 API group。 不要使用 "Input "和 "template.fn.crossplane.io"，而应使用对函数有意义的名称。

当你编辑 `input` 目录下的文件时，你必须通过运行 `go generate` 来更新一些已生成的文件。 详见 `input/generate.go`。

```shell
go generate ./...
```

{{</hint>}}

## 编辑模板，添加函数逻辑

您可以在{{<hover label="hello-world" line="1">}}运行函数{{</hover>}}方法中添加您的函数逻辑。 首次打开文件时，文件中包含一个 "hello world "函数。

```go {label="hello-world"}
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
    f.log.Info("Running Function", "tag", req.GetMeta().GetTag())

    rsp := response.To(req, response.DefaultTTL)

    in := &v1beta1.Input{}
    if err := request.GetInput(req, in); err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot get Function input from %T", req))
    	return rsp, nil
    }

    response.Normalf(rsp, "I was run with input %q", in.Example)
    return rsp, nil
}
```

所有 Go Composition 函数都有一个 "RunFunction "方法。 crossplane 会在一个{{<hover label="hello-world" line="1">}}RunFunctionRequest{{</hover>}}结构。

该函数通过返回一个{{<hover label="hello-world" line="13">}}RunFunctionResponse{{</hover>}}结构。

{{<hint "tip">}}crossplane 使用 [Protocol Buffers](http://protobuf.dev) 生成 `RunFunctionRequest` 和 `RunFunctionResponse` 结构。您可以在 [Buf Schema Registry](https://buf.build/crossplane/crossplane/docs/main:apiextensions.fn.proto.v1beta1) 中找到 `RunFunctionRequest` 和 `RunFunctionResponse` 的详细模式。{{</hint>}}

编辑 `RunFunction` 方法，将其替换为以下代码。

```go {hl_lines="4-56"}
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
    rsp := response.To(req, response.DefaultTTL)

    xr, err := request.GetObservedCompositeResource(req)
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot get observed composite resource from %T", req))
    	return rsp, nil
    }

    region, err := xr.Resource.GetString("spec.region")
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.region field of %s", xr.Resource.GetKind()))
    	return rsp, nil
    }

    names, err := xr.Resource.GetStringArray("spec.names")
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.names field of %s", xr.Resource.GetKind()))
    	return rsp, nil
    }

    desired, err := request.GetDesiredComposedResources(req)
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot get desired resources from %T", req))
    	return rsp, nil
    }

    _ = v1beta1.AddToScheme(composed.Scheme)

    for _, name := range names {
    	b := &v1beta1.Bucket{
    		ObjectMeta: metav1.ObjectMeta{
    			Annotations: map[string]string{
    				"crossplane.io/external-name": name,
    			},
    		},
    		Spec: v1beta1.BucketSpec{
    			ForProvider: v1beta1.BucketParameters{
    				Region: ptr.To[string](region),
    			},
    		},
    	}

    	cd, err := composed.From(b)
    	if err != nil {
    		response.Fatal(rsp, errors.Wrapf(err, "cannot convert %T to %T", b, &composed.Unstructured{}))
    		return rsp, nil
    	}

    	desired[resource.Name("xbuckets-"+name)] = &resource.DesiredComposed{Resource: cd}
    }

    if err := response.SetDesiredComposedResources(rsp, desired); err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot set desired composed resources in %T", rsp))
    	return rsp, nil
    }

    return rsp, nil
}
```

展开下面的代码块以查看完整的 `fn.go`，包括导入和解释函数逻辑的注释。

{{<expand "The full fn.go file" >}}

```go
package main

import (
    "context"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/utils/ptr"

    "github.com/upbound/provider-aws/apis/s3/v1beta1"

    "github.com/crossplane/function-sdk-go/errors"
    "github.com/crossplane/function-sdk-go/logging"
    fnv1beta1 "github.com/crossplane/function-sdk-go/proto/v1beta1"
    "github.com/crossplane/function-sdk-go/request"
    "github.com/crossplane/function-sdk-go/resource"
    "github.com/crossplane/function-sdk-go/resource/composed"
    "github.com/crossplane/function-sdk-go/response"
)

// Function returns whatever response you ask it to.
type Function struct {
    fnv1beta1.UnimplementedFunctionRunnerServiceServer

    log logging.Logger
}

// RunFunction observes an XBuckets composite resource (XR). It adds an S3
// bucket to the desired state for every entry in the XR's spec.names array.
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
    f.log.Info("Running Function", "tag", req.GetMeta().GetTag())

    // Create a response to the request. This copies the desired state and
    // pipeline context from the request to the response.
    rsp := response.To(req, response.DefaultTTL)

    // Read the observed XR from the request. Most functions use the observed XR
    // to add desired managed resources.
    xr, err := request.GetObservedCompositeResource(req)
    if err != nil {
    	// If the function can't read the XR, the request is malformed. This
    	// should never happen. The function returns a fatal result. This tells
    	// Crossplane to stop running functions and return an error.
    	response.Fatal(rsp, errors.Wrapf(err, "cannot get observed composite resource from %T", req))
    	return rsp, nil
    }

    // Create an updated logger with useful information about the XR.
    log := f.log.WithValues(
    	"xr-version", xr.Resource.GetAPIVersion(),
    	"xr-kind", xr.Resource.GetKind(),
    	"xr-name", xr.Resource.GetName(),
    )

    // Get the region from the XR. The XR has getter methods like GetString,
    // GetBool, etc. You can use them to get values by their field path.
    region, err := xr.Resource.GetString("spec.region")
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.region field of %s", xr.Resource.GetKind()))
    	return rsp, nil
    }

    // Get the array of bucket names from the XR.
    names, err := xr.Resource.GetStringArray("spec.names")
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.names field of %s", xr.Resource.GetKind()))
    	return rsp, nil
    }

    // Get all desired composed resources from the request. The function will
    // update this map of resources, then save it. This get, update, set pattern
    // ensures the function keeps any resources added by other functions.
    desired, err := request.GetDesiredComposedResources(req)
    if err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot get desired resources from %T", req))
    	return rsp, nil
    }

    // Add v1beta1 types (including Bucket) to the composed resource scheme.
    // composed.From uses this to automatically set apiVersion and kind.
    _ = v1beta1.AddToScheme(composed.Scheme)

    // Add a desired S3 bucket for each name.
    for _, name := range names {
    	// One advantage of writing a function in Go is strong typing. The
    	// function can import and use managed resource types from the provider.
    	b := &v1beta1.Bucket{
    		ObjectMeta: metav1.ObjectMeta{
    			// Set the external name annotation to the desired bucket name.
    			// This controls what the bucket will be named in AWS.
    			Annotations: map[string]string{
    				"crossplane.io/external-name": name,
    			},
    		},
    		Spec: v1beta1.BucketSpec{
    			ForProvider: v1beta1.BucketParameters{
    				// Set the bucket's region to the value read from the XR.
    				Region: ptr.To[string](region),
    			},
    		},
    	}

    	// Convert the bucket to the unstructured resource data format the SDK
    	// uses to store desired composed resources.
    	cd, err := composed.From(b)
    	if err != nil {
    		response.Fatal(rsp, errors.Wrapf(err, "cannot convert %T to %T", b, &composed.Unstructured{}))
    		return rsp, nil
    	}

    	// Add the bucket to the map of desired composed resources. It's
    	// important that the function adds the same bucket every time it's
    	// called. It's also important that the bucket is added with the same
    	// resource.Name every time it's called. The function prefixes the name
    	// with "xbuckets-" to avoid collisions with any other composed
    	// resources that might be in the desired resources map.
    	desired[resource.Name("xbuckets-"+name)] = &resource.DesiredComposed{Resource: cd}
    }

    // Finally, save the updated desired composed resources to the response.
    if err := response.SetDesiredComposedResources(rsp, desired); err != nil {
    	response.Fatal(rsp, errors.Wrapf(err, "cannot set desired composed resources in %T", rsp))
    	return rsp, nil
    }

    // Log what the function did. This will only appear in the function's pod
    // logs. A function can use response.Normal and response.Warning to emit
    // Kubernetes events associated with the XR it's operating on.
    log.Info("Added desired buckets", "region", region, "count", len(names))

    return rsp, nil
}
```

{{</expand>}}

此代码

1.从 `RunFunctionRequest` 获取观察到的 Composition 资源。
2.从观察到的 Composition 资源中获取区域和水桶名称。
3.为每个桶名添加一个所需的 S3 桶。
4.在 "RunFunctionResponse "中返回所需的 S3 存储桶。

代码使用了 [Upbound's AWS S3 Provider](https://github.com/upbound/provider-aws) 中的 `v1beta1.Bucket` 类型。用 Go 编写函数的一个好处是，你可以使用 crossplane 在其 Provider 中使用的强类型结构体来组成资源。

您必须获取 AWS Provider Go 模块才能使用这种类型: 

```shell
go get github.com/upbound/provider-aws@v0.43.0
```

crossplane 提供了一个[软件开发工具包](https://github.com/crossplane/function-sdk-go) (SDK)，用于在[Go](https://go.dev) 中编写组成函数。本函数被引用了 SDK 中的实用工具。特别是 `request` 和 `response` 包，使得使用 `RunFunctionRequest` 和 `RunFunctionResponse` 类型变得更加容易。

{{<hint "tip">}}请阅读 SDK 的 [Go package documentation](https://pkg.go.dev/github.com/crossplane/function-sdk-go)。{{</hint>}}

## 测试端到端功能

通过添加单元测试和被引用 "crossplane beta render "命令来测试你的函数。

Go 对单元测试有丰富的支持。 当你从模板初始化一个函数时，它会在 `fn_test.go`中添加一些单元测试。这些测试遵循 Go 的 [建议](https://github.com/golang/go/wiki/TestComments)。它们只引用 Go 标准库中的 [`pkg/testing`](https://pkg.go.dev/testing)和 [`google/go-cmp`](https://pkg.go.dev/github.com/google/go-cmp/cmp)。

要添加测试用例，请更新 `TestRunFunction` 中的 `cases` 映射。 展开下面的代码块，查看函数的完整 `fn_test.go` 文件。

{{<expand "The full fn_test.go file" >}}

```go
package main

import (
    "context"
    "testing"
    "time"

    "github.com/google/go-cmp/cmp"
    "github.com/google/go-cmp/cmp/cmpopts"
    "google.golang.org/protobuf/testing/protocmp"
    "google.golang.org/protobuf/types/known/durationpb"

    "github.com/crossplane/crossplane-runtime/pkg/logging"

    fnv1beta1 "github.com/crossplane/function-sdk-go/proto/v1beta1"
    "github.com/crossplane/function-sdk-go/resource"
)

func TestRunFunction(t *testing.T) {
    type args struct {
    	ctx context.Context
    	req *fnv1beta1.RunFunctionRequest
    }
    type want struct {
    	rsp *fnv1beta1.RunFunctionResponse
    	err error
    }

    cases := map[string]struct {
    	reason string
    	args args
    	want want
    }{
    	"AddTwoBuckets": {
    		reason: "The Function should add two buckets to the desired composed resources",
    		args: args{
    			req: &fnv1beta1.RunFunctionRequest{
    				Observed: &fnv1beta1.State{
    					Composite: &fnv1beta1.Resource{
    						// MustStructJSON is a handy way to provide mock
    						// resources.
    						Resource: resource.MustStructJSON(`{
    							"apiVersion": "example.crossplane.io/v1alpha1",
    							"kind": "XBuckets",
    							"metadata": {
    								"name": "test"
    							},
    							"spec": {
    								"region": "us-east-2",
    								"names": [
    									"test-bucket-a",
    									"test-bucket-b"
    								]
    							}
    						}`),
    					},
    				},
    			},
    		},
    		want: want{
    			rsp: &fnv1beta1.RunFunctionResponse{
    				Meta: &fnv1beta1.ResponseMeta{Ttl: durationpb.New(60 * time.Second)},
    				Desired: &fnv1beta1.State{
    					Resources: map[string]*fnv1beta1.Resource{
    						"xbuckets-test-bucket-a": {Resource: resource.MustStructJSON(`{
    							"apiVersion": "s3.aws.upbound.io/v1beta1",
    							"kind": "Bucket",
    							"metadata": {
    								"annotations": {
    									"crossplane.io/external-name": "test-bucket-a"
    								}
    							},
    							"spec": {
    								"forProvider": {
    									"region": "us-east-2"
    								}
    							}
    						}`)},
    						"xbuckets-test-bucket-b": {Resource: resource.MustStructJSON(`{
    							"apiVersion": "s3.aws.upbound.io/v1beta1",
    							"kind": "Bucket",
    							"metadata": {
    								"annotations": {
    									"crossplane.io/external-name": "test-bucket-b"
    								}
    							},
    							"spec": {
    								"forProvider": {
    									"region": "us-east-2"
    								}
    							}
    						}`)},
    					},
    				},
    			},
    		},
    	},
    }

    for name, tc := range cases {
    	t.Run(name, func(t *testing.T) {
    		f := &Function{log: logging.NewNopLogger()}
    		rsp, err := f.RunFunction(tc.args.ctx, tc.args.req)

    		if diff := cmp.Diff(tc.want.rsp, rsp, protocmp.Transform()); diff != "" {
    			t.Errorf("%s\nf.RunFunction(...): -want rsp, +got rsp:\n%s", tc.reason, diff)
    		}

    		if diff := cmp.Diff(tc.want.err, err, cmpopts.EquateErrors()); diff != "" {
    			t.Errorf("%s\nf.RunFunction(...): -want err, +got err:\n%s", tc.reason, diff)
    		}
    	})
    }
}
```

{{</expand>}}

使用 `go test` 命令运行单元测试: 

```shell
go test -v -cover .
=== RUN TestRunFunction
=== RUN TestRunFunction/AddTwoBuckets
--- PASS: TestRunFunction (0.00s)
    --- PASS: TestRunFunction/AddTwoBuckets (0.00s)
PASS
coverage: 52.6% of statements
ok github.com/negz/function-xbuckets 0.016s coverage: 52.6% of statements
```

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

使用 `go run` 在本地运行您的函数。

```shell {label="run"}
go run . --insecure --debug
```

{{<hint "warning">}}不安全 {{<hover label="run" line="1">}}不安全{{</hover>}}标志会告诉函数在不进行加密或身份验证的情况下运行。 只能在测试和开发过程中使用。{{</hint>}}

在另一个终端中，运行 `crossplane beta render`。

```shell
crossplane beta render xr.yaml composition.yaml functions.yaml
```

该命令调用你的函数。 在运行函数的终端中，现在应该可以看到 logging 输出: 

```shell
go run . --insecure --debug
2023-10-31T16:17:32.158-0700 INFO function-xbuckets/fn.go:29 Running Function        {"tag": ""}
2023-10-31T16:17:32.159-0700 INFO function-xbuckets/fn.go:125 Added desired buckets   {"xr-version": "example.crossplane.io/v1", "xr-kind": "XBuckets", "xr-name": "example-buckets", "region": "us-east-2", "count": 3}
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