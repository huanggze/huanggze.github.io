---
title: "K8s Operator 开发（二）：K8s API"
date: 2022-01-17T21:33:05+08:00
draft: true
toc: true
categories: ["Kubernetes"]
series: ["Kubernetes-operator-开发"]
---

## K8s API

Operator 通过与 kube-apiserver 通信访问 K8s 资源和自定义资源。kube-apiserver 是 Kubernetes 集群的核心组件和入口，暴露 RESTful HTTP API 接口，支持标准的 POST，GET，UPDATE，DELETE，PATCH 方法以及额外支持 WATCH[^1] 和 LIST 操作。以下是的例子，K8s API 文档可以参考 Kubernetes API Reference Docs[^2]。

```bash
# 创建 Deployment
POST /apis/apps/v1/namespaces/{namespace}/deployments

# 修改 Deployment
PATCH /apis/apps/v1/namespaces/{namespace}/deployments/{name}

# 监听 Nginx Deployment 的增删改事件通知
GET /apis/apps/v1/watch/namespaces/{namespace}/deployments?watch=1&fieldSelector=metadata.name=nginx
```

![k8s-operator-dev-part2-1](/images/k8s-operator-dev-part2-1.png)

大多数 Kubernetes 资源访问的 API 路径是 `/apis/{group}/{version}/namespaces/{namespace}/{resource}/{resourceName}`，比如 Deployment、Ingress 包括上一篇中的自定义资源 RebootPolicy，都有 API Group 和 Version。除了 K8s 早期的资源 Service、Node、Pod 由于历史原因，没有 API Group 只有版本（一般会称作 core API Group），路径是以 `/api/v1` 开头，如下图第二、三分支所示。这些常见的 K8s 数据类型都是版本化（**versioned**）、结构化（**structured**）。版本化指同一资源不同版本之间支持字段有差异；结构化指资源对象有对应的 Go 结构体，保证序列化和反序列化。URI 上使用 group、version、resource 来定位一个资源的形式称为 GVR，这和我们在 YAML 文件中使用的 apiVersion、kind 字段（又称 GVK 组合）是对应的。下一节我们会辨析 GVR、GVK 以及 Go Type 三者关系。

![k8s-operator-dev-part2-2](/images/k8s-operator-dev-part2-2.png)

### Unversioned Type

K8s 内部工作机制中也包含一些非版本化的数据类型（unversioned），通常用于服务发现等。比如，`kubectl api-resources` 返回 APIResourceList 列表。APIResourceList 在 K8s 内部就是一个非版本化类型，其他类型还包括 APIVersions、APIGroupList、Status（请求返回状态，如 kubectl get 一个不存在的 Deployment 返回的就是一个包含 404 信息的 Status），[详细信息见源码](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/apis/meta/v1/types.go)。

```bash
$ kubectl api-resources
NAME                              SHORTNAMES     APIVERSION                               NAMESPACED   KIND
bindings                                         v1                                       true         Binding
componentstatuses                 cs             v1                                       false        ComponentStatus
configmaps                        cm             v1                                       true         ConfigMap
endpoints                         ep             v1                                       true         Endpoints
events                            ev             v1                                       true         Event
limitranges                       limits         v1                                       true         LimitRange
namespaces                        ns             v1                                       false        Namespace
nodes                             no             v1                                       false        Node
...

$ curl http://<KUBERNETES_HOST>:<KUBERNETES_PORT>/apis
{
  "kind": "APIGroupList",
  "apiVersion": "v1", # 为了方便统一，K8s 代码实现里会视作 unversioned 类型的版本为 v1
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        },
        {
          "groupVersion": "apiregistration.k8s.io/v1beta1",
          "version": "v1beta1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
    },
    ...
  ]
}
```

> 注： 为了方便统一，K8s 代码实现里会视作 unversioned 类型的版本为 v1。源码：k8s.io/apimachinery/pkg/apis/meta/v1/register.go
> ```go
> // Unversioned is group version for unversioned API objects
> // TODO: this should be v1 probably
> var Unversioned = schema.GroupVersion{Group: "", Version: "v1"}
> ```

### Unstructured Type

非结构化类型主要用于 K8s 扩展等，支持定义由其他外部插件来编解码。这样的数据类型为 `map[string]interface{}`，而不是一个具体的 Go 结构体。目前暂时没有发现 unstructured 类型的典型用例。

```go
// 源码：k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go

// Unstructured allows objects that do not have Golang structs registered to be manipulated 
// generically. This can be used to deal with the API objects from a plug-in. Unstructured
// objects still have functioning TypeMeta features-- kind, version, etc.

type Unstructured struct {
	// Object is a JSON compatible map with string, float, int, bool, []interface{}, or
	// map[string]interface{}
	// children.
	Object map[string]interface{}
}
```

## Go Type、GVK 与 GVR

当我们需要和 K8s 资源打交道时，我们需要搞清楚资源在 K8s 内部的表现形式。有三个概念很重要：Go Type，GVK，GVR。Go Type 是资源的 Go 语言类型，用于序列化和反序列化。GVK（Group + Version + Kind）唯一标识一种资源的资源类型。GVR 是 URI 中的概念。当我们要操作 K8s 资源时，需要依靠 GVK 和 GVR 拼接请求路径，Go Type 序列化/反序列化请求体和返回体，通过底层 http.Client 发送请求。

> GVK 和 GVR 是 k8s.io/apimachinery 包里的两个重要概念，区别是 GVK 代表一个 Object 类型，而 GVR 代表一个 HTTP 请求 Path。

### Go Type

结构化 K8s 资源都声明有自己的 Go 结构体。比如自定义资源 RebootPolicy，Go 代码定义在 api/v1alpha1/rebootpolicy_types.go 目录下。K8s 资源的类型声明源码地址在 k8s.io/api 包下各自的 /{group}/{version}/types.go 中。如下是 Deployment 类型的声明[源码](https://github.com/kubernetes/api/blob/v0.23.0/apps/v1/types.go#L318)。结构化的 K8s 资源都实现了 [runtime.Object 接口](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/runtime/interfaces.go#L302)。

```go
// Deployment enables declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

### GVK

[GVK](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/runtime/schema/group_version.go#L142) 是 K8s 资源运转调度的核心。GVK 数据类型定义为 k8s.io/apimachinery/pkg/runtime/schema 包下的 `GroupVersionKind` 类型：

```go
// GroupVersionKind unambiguously identifies a kind.  It doesn't anonymously include GroupVersion
// to avoid automatic coercion.  It doesn't use a GroupVersion to avoid custom marshalling
type GroupVersionKind struct {
	Group   string
	Version string
	Kind    string
}
```

Go Type 和 GVK 的双向映射是通过在 [Scheme 对象](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/runtime/scheme.go#L46)中注册实现。使用 scheme 可以辅助快速定位到，这在 Kubebuilder 中大量使用。你基本上可以看到所有的 client 结构体中都有 Scheme 的身影。Scheme 的功能总结如下：

1. xxx

```go
type Scheme struct {
	gvkToType map[schema.GroupVersionKind]reflect.Type
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	unversionedTypes map[reflect.Type]schema.GroupVersionKind
	unversionedKinds map[string]reflect.Type

	converter *conversion.Converter
}
```

### GVR

[GVR](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/runtime/schema/group_version.go#L96) 也需要与 GVK 建立映射关系（再一次看到 GVK 作为 K8s 资源运转调度的核心重要地位）。GVR 和 GVK 的双向映射放在 [RESTMapping](https://github.com/kubernetes/apimachinery/blob/v0.23.0/pkg/api/meta/interfaces.go#L93) 对象中。GVR 会在组装 HTTP 请求路径时用到，[详见源码](https://github.com/kubernetes/client-go/blob/v0.23.1/rest/request.go#L470-L476)。

```go
// RESTMapping contains the information needed to deal with objects of a specific
// resource and kind in a RESTful manner.
type RESTMapping struct {
	// Resource is the GroupVersionResource (location) for this endpoint
	Resource schema.GroupVersionResource

	// GroupVersionKind is the GroupVersionKind (data format) to submit to this endpoint
	GroupVersionKind schema.GroupVersionKind

	// Scope contains the information needed to deal with REST Resources that are in a resource hierarchy
	Scope RESTScope
}
```

## Client 库

discoveryClient:  discovery.NewDiscoveryClientForConfig(cfg)


dynamic 是指运行时，和static 相对
func NewDynamicRESTMapper(
drm.setStaticMapper();

创建 rest config
client.apiutil
func createRestConfig

encoder, err := r.c.content.Negotiator.Encoder(r.c.content.ContentType, nil)
Negotiator 进行编码

SpecificallyVersionedParams 是指 QueryParam


https://zzl-code.github.io/cloudNative/k8s_client_go



[^1]: [Kubernetes API Concepts: Efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)

[^2]: [Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23)